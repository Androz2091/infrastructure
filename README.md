todo sylvain:
- version de k8s

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

* `sudo apt update && sudo apt upgrade -y`

Turn off swap.

```sh
systemctl mask dev-sdb?.swap && systemctl stop dev-sdb?.swap # Debian special, check dans htop`
```

Install CRI-O.

```sh
export VERSION=1.30
export OS=Debian_12

echo 'deb http://deb.debian.org/debian buster-backports main' > /etc/apt/sources.list.d/backports.list
apt update
apt install -y -t buster-backports libseccomp2 || apt update -y -t buster-backports libseccomp2

echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb [signed-by=/usr/share/keyrings/libcontainers-crio-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

mkdir -p /usr/share/keyrings
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | gpg --dearmor -o /usr/share/keyrings/libcontainers-archive-keyring.gpg
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | gpg --dearmor -o /usr/share/keyrings/libcontainers-crio-archive-keyring.gpg

apt-get update
apt-get install -y cri-o cri-o-runc
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

Install kubeadm, kubelet and kubectl.

```sh
apt-get update
apt-get install -y apt-transport-https ca-certificates curl

# Google Cloud public signing key
mkdir /etc/apt/keyrings
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

# Kubernetes apt repo
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl

# prevent the package from being automatically installed, upgraded or removed.
apt-mark hold kubelet kubeadm kubectl
```

Configure kubectl CLI to connect to the cluster.

* `export KUBECONFIG=/etc/kubernetes/admin.conf`

Allow the current (single) node to be a worker node.

* `kubectl taint nodes --all node-role.kubernetes.io/control-plane-`

### Install a CNI plugin

* `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`

### Install Caddy

* `sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl`
* `curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg`
* `curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list`
* `sudo apt update`
* `sudo apt install caddy`

Start Caddy (execute this command in the directory where the `Caddyfile` is located).

* `sudo caddy start`

### Install Helm

* `curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -`
* `sudo apt-get install apt-transport-https --yes`
* `echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list`
* `sudo apt-get update`
* `sudo apt-get install helm`

### Install Sealed Secrets

* `helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets`
* `helm repo update`
* `helm install sealed-secrets sealed-secrets/sealed-secrets --namespace kube-system --create-namespace --version 2.16.1`

Export the public key.

* `kubeseal --fetch-cert --controller-name=sealed-secrets --controller-namespace=kube-system > sealed-secrets.crt`

### Install ArgoCD

* `kubectl create namespace argocd`
* `helm repo add argo https://argoproj.github.io/argo-helm`
* `helm repo update`
* `helm install argocd argo/argo-cd --namespace argocd --create-namespace --values https://raw.githubusercontent.com/Androz2091/k8s-infrastructure/main/argocd-values.yaml --version 7.0.0`

CLI de argo.

* `sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 sudo chmod +x /usr/local/bin/argocd`

Get ArgoCD password.

* `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`

### Install Longhorn

* `helm repo add longhorn https://charts.longhorn.io`
* `helm repo update`
* `helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.7.0`

(optional) forward the Longhorn UI to the host.

`kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80`

todo /var/lib/longhorn

### Execute bootstrap application

* `kubectl apply -f https://raw.githubusercontent.com/Androz2091/k8s-infrastructure/main/bootstrap-app.yaml`

### Use k8s cluster DNS on the host

* update `/etc/resolv.conf` as follows:
```sh
nameserver 10.96.0.10
```

Now we also need to update the cluster DNS so it does not loop back to the host.

* dump the current CoreDNS config:
`kubectl -n kube-system get configmap coredns -o yaml > coredns_patched_dns.yaml`

* edit the `coredns_patched_dns.yaml` file and add the following line to the `Corefile`:
```sh
forward . 1.1.1.1 8.8.8.8 {
	max_concurrent 1000
}
```

* then apply the changes by running:
`kubectl apply -f coredns_patched_dns.yaml`

### 

* `kubeseal --scope namespace-wide --cert ../../../sealed-secrets.crt -o yaml < secrets.yaml > sealed-secrets.yaml`

### Changer le DNS

### Se co à la db dans PgAdmin

Ajouter `postgresql.db.svc.cluster.local`

