# K8S infrastructure

## Setup

### Create the k8s cluster

* `sudo apt update && sudo apt upgrade -y`

Turn off swap.

* `sudo swapoff -a`
* `sudo sed -i '/ swap / s/^/#/' /etc/fstab`

Install CRI-O and kubeadm.

* `export KUBERNETES_VERSION=v1.30`
* `export CRIO_VERSION=v1.30`
* `apt-get install -y software-properties-common curl`
* `curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`
* `echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list`
* `curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg`
* `echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" | tee /etc/apt/sources.list.d/cri-o.list`
* `sudo apt-get update`
* `sudo apt-get install -y cri-o kubelet kubeadm kubectl`
* `sudo systemctl start crio.service`

todo ask sylvain

* `modprobe br_netfilter`
* `sysctl -w net.ipv4.ip_forward=1`
* `kubeadm init`

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

### Install ArgoCD

* `kubectl create namespace argocd`
* `helm repo add argo https://argoproj.github.io/argo-helm`
* `helm repo update`
* `helm install argocd argo/argo-cd --namespace argocd --create-namespace --values https://raw.githubusercontent.com/Androz2091/k8s-infrastructure/main/argocd-values.yaml --version 7.0.0`

todo sylvain is this needed?
* `sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 sudo chmod +x /usr/local/bin/argocd`

Get ArgoCD password.

* `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`

### Install Longhorn

* `helm repo add longhorn https://charts.longhorn.io`
* `helm repo update`
* `helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.7.0`

(optional) forward the Longhorn UI to the host.

`kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80`

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

Useful commands:

* `kubeseal --scope namespace-wide --cert ../../../sealed-secrets.crt -o yaml < secrets.yaml > sealed-secrets.yaml`

### Changer le DNS

### Se co à la db dans PgAdmin

Ajouter `postgresql.db.svc.cluster.local`

## Namespaces

### DB

* PostgreSQL
* PgAdmin4

### Managed

This namespace is for all the services hosted for my customers (mainly Discord bots but also websites, APIs, etc.).

* 10 Mans Discord Bot
* Androz Development Bot
* Blockfella Discord Bot
* Discordsopli Discord Bot
* CustomGPT Discord Bot
* Evolution Markets Discord Bot
* Fractal Markets Discord Bot
* KOLC Discord Bot
* Kryptview Discord Bot
* Kryptview Community Discord Bot
* Lapiz Legion Discord Bot
* Music Cat Discord Bot
* Netflix Discord Bot
* Pokercode Discord Bot
* Pokercode Quiz Discord Bot
* SimpleTrading Community Discord Bot
* TPFisher Discord Bot
* WhatIsLife Discord Bot
* Wheel Discord Bot

### Pro

This namespace is for all the services hosted for me as a freelancer.

* Umami
* HasteServer
* Blog
* DDPE
* Diswho
* Instaddict

### ManageInvite

* ManageInvite API
* ManageInvite Dashboard
* ManageInvite Bot

### Home

* Vaultwarden ✅
* TimeTagger ✅
* Immich
* Monica
* Mealie
* FileBrowser ✅

### Dumpus

⚠️ requires extra network isolation for security reasons ⚠️  
how to do so...?

* Dumpus API
* some Dumpus workers (can we make it auto scale?)

### Nextcloud

This namespace is for all the services related to Nextcloud.

### Sushiflix

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
