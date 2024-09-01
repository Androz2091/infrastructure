# K8S infrastructure

## Introduction

This repository contains the infrastructure configuration for my Kubernetes cluster.

Table of contents:
* [🚀 Apps](#-Apps)
* [📜 Wiki](#-Wiki)
* [⚒️ Setup](#-Setup)

## 🚀 Apps

### Namespace DB

* PostgreSQL
* PgAdmin4

### Namespace Managed

This namespace is for all the services hosted for my customers (mainly Discord bots but also websites, APIs, etc.).

### Namespace Pro

This namespace is for all the services hosted for me as a freelancer.

* Umami
* HasteServer
* Blog
* DDPE
* Diswho
* Instaddict

### Namespace ManageInvite

* ManageInvite API
* ManageInvite Dashboard
* ManageInvite Bot

### Namespace Home

* Vaultwarden ✅
* TimeTagger ✅
* Immich
* Monica
* Mealie
* FileBrowser ✅

### Namespace Dumpus

⚠️ requires extra network isolation for security reasons ⚠️  
how to do so...?

* Dumpus API
* some Dumpus workers (can we make it auto scale?)

### Namespace Nextcloud

This namespace is for all the services related to Nextcloud.

### Namespace Sushiflix

This namespace is for all media services.

* Plex
* Radarr
* Sonarr
* Bazarr
* Jackett
* Qbittorrent
* Sabnzbd
* Tautulli
* Overseerr

## 📜 Wiki

See https://wiki2.agepoly.ch/.

### Create a new sealed secret

Create a new secret (the simplest way is to use `stringData`).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secret
  namespace: {NAMESPACE_NAME} # IMPORTANT
type: Opaque
stringData:
  username: martin007
  password: mOnSuper1M0t2pass
  code-secret: "1234 5678 9101"
```

Encrypt the secret.

* `kubeseal --scope namespace-wide --cert https://raw.githubusercontent.com/Androz2091/k8s-infrastructure/main/sealed-secrets.crt -o yaml < secrets.yaml > sealed-secrets.yaml`

### Port forward (to a service or a pod)

* `kubectl -n somenamespace port-forward svc/someservice host-port:cluster-port`

### Preview manifests created by Helm charts 

* `helm template my-app repo-url/app -f values.yaml`

Same applies for `kustomization.yaml` files:

* `kubectl kustomize --enable-helm .`

## ⚒️ Setup

### Create the k8s cluster

```sh
apt update && sudo apt upgrade -y
apt-get install -y software-properties-common curl
```

Turn off swap.

```sh
swapoff -a
systemctl mask dev-sdb?.swap && systemctl stop dev-sdb?.swap # Debian special, check dans htop`
```

Install CRI-O and Kubernetes. See [cri-o/packaging instructions.](https://github.com/cri-o/packaging/blob/main/README.md#distributions-using-deb-packages).

```sh
KUBERNETES_VERSION=v1.31
CRIO_VERSION=v1.30
```

Add the Kubernetes repository.

```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
```

Add the CRI-O repository.

```sh
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list
```

Install the packages.

```sh
apt-get update
apt-get install -y cri-o kubelet kubeadm kubectl
apt-mark hold cri-o kubelet kubeadm kubectl
```

Start the cluster

```sh
systemctl start crio.service
```

Forwarding IPv4 and letting iptables see bridged traffic.

```sh
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sysctl --system

# Checks
lsmod | grep br_netfilter
lsmod | grep overlay

systemctl enable --now crio
```

Create the cluster.

```sh
kubeadm init --pod-network-cidr=10.244.0.0/16
```

Configure kubectl CLI to connect to the cluster.

```sh
export KUBECONFIG=/etc/kubernetes/admin.conf
```

Allow the current (single) node to be a worker node.

```sh
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Install a CNI plugin

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### Install Caddy

```sh
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

Start Caddy (execute this command in the directory where the `Caddyfile` is located).

```sh
sudo caddy start
```

### Install Helm

```sh
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Install Sealed Secrets

```sh
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets sealed-secrets/sealed-secrets --namespace kube-system --create-namespace --version 2.16.1
```

Export the public key.

```sh
kubeseal --fetch-cert --controller-name=sealed-secrets --controller-namespace=kube-system > sealed-secrets.crt
```

### Install ArgoCD

```sh
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd --create-namespace --values https://raw.githubusercontent.com/Androz2091/k8s-infrastructure/main/argocd-values.yaml --version 7.0.0
```

CLI de argo.

```sh
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 sudo chmod +x /usr/local/bin/argocd
```

Get ArgoCD password.

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### Install Longhorn

```sh
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.7.0
```

(optional) forward the Longhorn UI to the host.

```sh
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```

todo /var/lib/longhorn

### Execute bootstrap application

```sh
kubectl apply -f https://raw.githubusercontent.com/Androz2091/k8s-infrastructure/main/bootstrap-app.yaml
```

### Use k8s cluster DNS on the host

* update `/etc/resolv.conf` as follows:
```sh
nameserver 10.96.0.10
```

Now we also need to update the cluster DNS so it does not loop back to the host.

* dump the current CoreDNS config:
```sh
kubectl -n kube-system get configmap coredns -o yaml > coredns_patched_dns.yaml
```

* edit the `coredns_patched_dns.yaml` file and add the following line to the `Corefile`:
```sh
forward . 1.1.1.1 8.8.8.8 {
	max_concurrent 1000
}
```

* then apply the changes by running:
```sh
kubectl apply -f coredns_patched_dns.yaml
```

### 

```sh
kubeseal --scope namespace-wide --cert ../../../sealed-secrets.crt -o yaml < secrets.yaml > sealed-secrets.yaml
```

### Changer le DNS

### Se co à la db dans PgAdmin

Ajouter `postgresql.db.svc.cluster.local`

