# Kubernetes API Server Installation (Updated)

## Pre-requisite

```bash
cd /root/binaries/kubernetes/server/bin/
sudo cp kube-apiserver /usr/local/bin/

echo $SERVER_IP
```

---

## Step 1: Generate the API Server OpenSSL Configuration

```bash
cd /root/certificates

cat <<EOF > api.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local

IP.1 = 127.0.0.1
IP.2 = ${SERVER_IP}
IP.3 = 10.32.0.1
EOF
```

## Step 2: Generate the API Server Certificate

```bash
openssl genrsa -out kube-api.key 2048

openssl req \
  -new \
  -key kube-api.key \
  -subj "/CN=kube-apiserver" \
  -out kube-api.csr \
  -config api.conf

openssl x509 \
  -req \
  -in kube-api.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out kube-api.crt \
  -extensions v3_req \
  -extfile api.conf \
  -days 3650
```

## Step 3: Generate the Service Account Certificate

```bash
openssl genrsa -out service-account.key 2048

openssl req \
  -new \
  -key service-account.key \
  -subj "/CN=service-accounts" \
  -out service-account.csr

openssl x509 \
  -req \
  -in service-account.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out service-account.crt \
  -days 3650
```

## Step 4: Copy Certificates

```bash
sudo mkdir -p /var/lib/kubernetes

sudo cp \
  ca.crt \
  etcd.crt \
  etcd.key \
  kube-api.crt \
  kube-api.key \
  service-account.crt \
  service-account.key \
  /var/lib/kubernetes/

ls -l /var/lib/kubernetes
```

## Step 5: Configure Encryption at Rest

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat <<EOF > encryption-at-rest.yaml
kind: EncryptionConfig
apiVersion: apiserver.config.k8s.io/v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

sudo cp encryption-at-rest.yaml /var/lib/kubernetes/
```

## Step 6: Create the systemd Service

Create `/etc/systemd/system/kube-apiserver.service`:

```ini
[Unit]
Description=Kubernetes API Server
Documentation=https://kubernetes.io/docs/
After=network.target etcd.service
Requires=etcd.service

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${SERVER_IP} \
  --bind-address=0.0.0.0 \
  --secure-port=6443 \
  --allow-privileged=true \
  --anonymous-auth=false \
  --authorization-mode=Node,RBAC \
  --client-ca-file=/var/lib/kubernetes/ca.crt \
  --enable-admission-plugins=NodeRestriction,NamespaceLifecycle,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --enable-bootstrap-token-auth=true \
  --etcd-cafile=/var/lib/kubernetes/ca.crt \
  --etcd-certfile=/var/lib/kubernetes/etcd.crt \
  --etcd-keyfile=/var/lib/kubernetes/etcd.key \
  --etcd-servers=https://127.0.0.1:2379 \
  --kubelet-client-certificate=/var/lib/kubernetes/kube-api.crt \
  --kubelet-client-key=/var/lib/kubernetes/kube-api.key \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kube-api.crt \
  --tls-private-key-file=/var/lib/kubernetes/kube-api.key \
  --service-account-key-file=/var/lib/kubernetes/service-account.crt \
  --service-account-signing-key-file=/var/lib/kubernetes/service-account.key \
  --service-account-issuer=https://${SERVER_IP}:6443 \
  --encryption-provider-config=/var/lib/kubernetes/encryption-at-rest.yaml \
  --audit-log-path=/var/log/kube-apiserver-audit.log \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --event-ttl=1h \
  --profiling=false \
  --v=2

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Step 7: Reload and Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver
sudo systemctl start kube-apiserver
sudo systemctl status kube-apiserver
```

Verify:

```bash
sudo ss -lntp | grep 6443
```

Logs:

```bash
sudo journalctl -u kube-apiserver -f
```
