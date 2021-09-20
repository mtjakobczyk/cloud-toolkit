
Executed on 20.09.2021 based on [Kubernetes The Hard Way 1.20.0](https://github.com/kelseyhightower/kubernetes-the-hard-way/tree/1.21.0).

## 1. Client configuration
Download these utilities:
- gcloud
- kubectl
- cfssl

Initialize `gcloud`:
```bash
gcloud init
# or just: gcloud auth login
```
You will be prompted to login to the Google Account using a web browser and agree to authorize `gcloud` access to GCP APIs.
The access token and refresh token will be stored in `access_tokens.db` and `credentials.db` here: `~/.config/gcloud`

Set the default project, region and zone:
```bash
# list projects with: gcloud projects list
PROJECT_ID=michals-kubernetes-platform
REGION=europe-central2
ZONE=europe-central2-b
gcloud config set project $PROJECT_ID
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
```
These settings are stored in `~/.config/gcloud/configurations/config_default`.


## 2. Infrastructure
Create a Virtual Network (`mk8s-network`) for all Kubernetes Nodes. Subnet Mode `custom` means no subnets are created along with the virtual network in the beginning. We create one subnet (`kubernetes-mk8s-subnet`) manually. 
```bash
CLUSTER=mk8s
VNET=$CLUSTER-network
SUBNET=kubernetes-$CLUSTER-subnet
gcloud compute networks create $VNET --subnet-mode custom
gcloud compute networks subnets create $SUBNET \
  --network $VNET \
  --range 10.240.0.0/24

# Firewall rule to allow internal communication across all protocols
gcloud compute firewall-rules create $VNET-allow-internal \
  --allow tcp,udp,icmp \
  --network $VNET \
  --source-ranges 10.240.0.0/24,10.200.0.0/16

# Firewall rule to allow external SSH, ICMP, and HTTPS
gcloud compute firewall-rules create $VNET-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network $VNET \
  --source-ranges 0.0.0.0/0

# Static IP which will be attached to the external load balancer fronting the Kube API
gcloud compute addresses create $CLUSTER \
  --region $(gcloud config get-value compute/region)
```
Provision compute instances
```bash
# Control Plane
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet $SUBNET \
    --tags kubernetes-$CLUSTER,controller
done
# Worker Nodes
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet $SUBNET \
    --tags kubernetes-$CLUSTER,worker
done
```

## 3. Certificates
**Goal:** Generate TLS certificates and distribute them to nodes or embed them into kubernetes configuration files

Use [Cloudflare's PKI and TLS toolkit](https://github.com/cloudflare/cfssl) to: 
1. generate a self-signed root CA certificate and 
2. generate certificates for:
    - etcd
    - kube-apiserver
    - kube-controller-manager
    - kube-scheduler
    - kubelet
    - kube-proxy

### 3.1 Self-signed root CA certificate
```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "PL",
      "L": "Warsaw",
      "O": "Kubernetes",
      "OU": "CA"
    }
  ]
}
EOF

# Generate self-signed root CA certificate
mkdir ~/workspaces/$CLUSTER & cd ~/workspaces/$CLUSTER
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
This produces two `ca`-relevant PEM-encoded entities:
```bash
ca.pem # the self-signed certificate for the CA
ca-key.pem # the corresponding private key
```

### 3.2 The Admin Client Certificate
The predefined `system:masters` group is a group which is hardcoded into the Kubernetes API server source code as having unrestricted rights to the Kubernetes API server. Any user who is a member of this group has full cluster-admin rights to the cluster. ([Source](https://blog.aquasec.com/kubernetes-authorization)).

The cluster will have Kubernetes client certificate authentication model enabled. The holder of this certificate will be identified as the `system:masters` group member and will be able to enjoy the full access to Kube API.
```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "PL",
      "L": "Warsaw",
      "O": "system:masters"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```
This produces:
```bash
admin.pem # the certificate for the admin user
admin-key.pem # the corresponding private key
```

### 3.3 The Kubelet Client Certificates
Kubernetes nodes execute Kubelet daemons. The Kubelets make requests to the Kube API. The requests are authorized by the [Node Authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/).

> Kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`. 
> The value of `<nodeName>` must match precisely the name of the node as registered by the kubelet.

For each Kubernetes worker node, create a certificate that meets the Node Authorizer requirements:
```bash
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "PL",
      "L": "Warsaw",
      "O": "system:nodes"
    }
  ]
}
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```
This produces:
```bash
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### 3.4 The Controller Manager Client Certificate
```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "PL",
      "L": "Warsaw",
      "O": "system:kube-controller-manager"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```
### 3.5 The Kube Proxy Client Certificate
```bash
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "PL",
      "L": "Warsaw",
      "O": "system:node-proxier"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```
### 3.6 The Scheduler Client Certificate
```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "PL",
      "L": "Warsaw",
      "O": "system:kube-scheduler"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```
### 3.7 The Kubernetes API Server Certificate
```bash
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe $CLUSTER \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "PL",
      "L": "Warsaw",
      "O": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```
- The `10.32.0.1` is the first IP address from the **Services CIDR range** we would set later to `10.32.0.0/24`. This first IP will link to the name `kubernetes.default.svc.cluster.local` ([Source](https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/105)).
- The three IP addresses `10.240.0.10`, `10.240.0.11`, `10.240.0.12` are the virtual-network-internal private IPs of the controllers.

This produces:
```
kubernetes-key.pem
kubernetes.pem
```

### 3.8 The Service Account Key Pair
```bash
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "PL",
      "L": "Warsaw",
      "O": "Kubernetes"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

### 3.9 Distributing certificates to compute instances
Upload the certificates and keys to the worker nodes:
```bash
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```

Upload the certificates and keys to the controller nodes:
```bash
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

## 4. Kubernetes configuration files
**Goal:** Generate Kubernetes configuration files and distribute them to nodes.

A Kubernetes configuration file (aka *kubeconfig*) includes settings that point to an IP address or a resolvable DNS name of a load balancer fronting the Kubernetes API.
```bash
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe $CLUSTER \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

### 4.1 Kubernetes configuration files for worker node Kubelets
For each worker node Kubelet we will create a kubeconfig file by performing 4 steps:
1. use the `set-cluster` command to specify the Kube API URL (`--server`) and embed the CA certificate (`--certificate-authority`) used for Kube API
2. use the `set-credentials` command to embed the certificate (`--client-certificate`) and key (`--client-key`) generated for Kubelet
3. use the `set-context` command to combine both cluster and credentials into one context
```bash
for instance in worker-0 worker-1 worker-2; do
  kubectl config set-cluster $CLUSTER \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=$CLUSTER \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
As a result, each worker node Kubelet will have its dedicated `kubeconfig`:
```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### 4.2 Kubernetes configuration files for kube-proxy 
```bash
kubectl config set-cluster $CLUSTER \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=$CLUSTER \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
produces:
```
kube-proxy.kubeconfig
```
### 4.3 Kubernetes configuration files for kube-controller-manager 
```bash
kubectl config set-cluster $CLUSTER \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=$CLUSTER \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```
produces:
```
kube-controller-manager.kubeconfig
```

### 4.4 Kubernetes configuration files for kube-scheduler 
```bash
kubectl config set-cluster $CLUSTER \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=$CLUSTER \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```
produces:
```
kube-scheduler.kubeconfig
```

### 4.5 Kubernetes configuration files for the admin user
```bash
kubectl config set-cluster $CLUSTER \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=$CLUSTER \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```
produces:
```
admin.kubeconfig
```

### 3.9 Distributing kubeconfig files to compute instances
Upload the kubeconfig files to the worker nodes:
```bash
for instance in worker-0 worker-1 worker-2; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

Upload the kubeconfig files to the controller nodes:
```bash
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

### 4 Data at rest encryption key
**Goal:** Generate data-at-rest encryption key and encapsulate it in an `EncryptionConfig` resource YAML
```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
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
```

Upload the `EncryptionConfig` resource YAML to the controller nodes:
```bash
for instance in controller-0 controller-1 controller-2; do
  gcloud compute scp encryption-config.yaml ${instance}:~/
done
```

## X. Cleanup
To reverse the step **2. Infrastructure** we basically need to destroy the IaaS resources:
```bash
# TODO
```