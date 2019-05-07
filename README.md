# k8s Cluster Setup Notes

Provision three or more servers. Master node server should have a minimum of 2 CPUs and 2G of RAM.

Use Ubuntu 18.10 and update to latest packages.

```bash
apt-get upgrade -y
```

## Private Networking

Setup private network interfaces if configured server instances to do already have them. See providers documentation for private network support.

The following configures a network interface for the private IP address **10.5.96.3**.

```bash
cat > /etc/netplan/10-ens7.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ens7:
      mtu: 1450
      dhcp4: no
      addresses: [10.5.96.7/20]
EOF
```

Apply the network settings:
```bash
netplan apply
```

Ensure the interface **ens7** is up:
```bash
ifconfig
```

Configure private network addresses for each server in the cluster and ensure they can ping each other over the private interfaces.


## Install Docker

```bash
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
    
```

Add Docker's GPG Key:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Add Docker's Repository:
```bash
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

Update package list:
```bash
apt-get update
```

Install Docker
```bash
apt-get install docker-ce docker-ce-cli containerd.io
```

Configure Docker Daemon (native.cgroupdriver=systemd):
```bash
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

```bash
mkdir -p /etc/systemd/system/docker.service.d
```

Restart Docker:
```bash
systemctl daemon-reload && systemctl restart docker
```

## Install Kubernetes

Install **kubelet**, **kubeadm** and **kubectl**.

Add Google GPG key for apt:
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
``` 

Add Kubernetes apt repository:
```bash
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

Update list of packages:
```bash
apt-get update
```

Install packages:
```bash
apt-get install -y kubelet kubeadm kubectl
```

Keep packages from being updated:
```bash
apt-mark hold kubelet kubeadm kubectl
```

## Install Master Node

Advertise the API server to other nodes on the private IP. Instruct Kubernetes to include the public IP in the certificate for remote access from the public network.
```bash
kubeadm init --apiserver-advertise-address=10.5.96.3 --apiserver-cert-extra-sans=45.63.55.184
```

### Create Overlay Network

```bash
cat <<EOF >/etc/systemd/system/overlay-route.service
[Unit]
Description=Route weave through private network.

[Service]
Type=oneshot
User=root
ExecStart=/sbin/ip route add 10.96.0.0/16 dev ens7 src 10.5.96.7

[Install]
WantedBy=multi-user.target
EOF
```

Enable overlay network:
```bash
systemctl enable overlay-route.service
```

### Install Pod Network (Weave)

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

## Join Node to Cluster

```bash
kubeadm join 10.5.96.3:6443 --token 5...sa     --discovery-token-ca-cert-hash sha256:6f...9c --apiserver-advertise-address=10.5.96.5 
```

## Test

```bash
kubectl run --generator=run-pod/v1 -it --rm  cli --image=tutum/curl:alpine --command -- sh
```