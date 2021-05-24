# Kubernetes bootstrap with Kubeadm on GCP and Ubuntu 20.04

Example for manual cluster bootstrap from scratch, was tested on v1.20

Was inspired by official doc and the following articles:    
https://kvaps.medium.com/creating-high-available-baremetal-kubernetes-cluster-with-kubeadm-and-keepalived-simplest-guide-71766d5e25ae    
https://kvaps.medium.com/for-make-this-scheme-more-safe-you-can-add-haproxy-layer-between-keepalived-and-kube-apiservers-62c344283076    

## GCP resources creation

- Create VPC:

```
gcloud compute networks create k8s-vpc --subnet-mode custom
```

- Create Subnet:

```
gcloud compute networks subnets create k8s-subnet \
  --network k8s-vpc \
  --range 192.168.0.0/24
```

- Create Firewall rules:

```
gcloud compute firewall-rules create k8s-internal \
  --network k8s-vpc \
  --allow tcp,udp,icmp \
  --source-ranges 192.168.0.0/24

gcloud compute firewall-rules create k8s-external \
  --allow tcp:22,tcp:6443,tcp:3389,icmp \
  --network k8s-vpc \
  --source-ranges 35.235.240.0/20

gcloud compute firewall-rules list --filter="network:k8s-vpc"
```

- Configure IAP tunneling:

```
gcloud projects add-iam-policy-binding reliable-fort-313713 \
    --member=user:drenora@gmail.com \
    --role=roles/iap.tunnelResourceAccessor

gcloud compute routers create nat-router \
    --network k8s-vpc \
    --region asia-east1

gcloud compute routers nats create nat-config \
    --router-region asia-east1 \
    --router nat-router \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```

- Create Compute Instances

```
gcloud compute instances create k8s-lb \
  --async \
  --boot-disk-size 10GB \
  --can-ip-forward \
  --image-family ubuntu-1804-lts \
  --image-project ubuntu-os-cloud \
  --machine-type e2-medium \
  --private-network-ip 192.168.0.10 \
  --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
  --subnet k8s-subnet \
  --tags kubernetes-test,load-balancer \
  --no-address

for i in 1 2 3; do
  gcloud compute instances create master-${i} \
    --async \
    --boot-disk-size 100GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-medium \
    --private-network-ip 192.168.0.$(($i + 10)) \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --tags kubernetes-test,master \
    --no-address
done

for i in 1 2 3; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 100GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-medium \
    --private-network-ip 192.168.0.$(($i + 20)) \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-subnet \
    --tags kubernetes-test,worker \
    --no-address
done

gcloud compute instances list
```

## LB node preparation

- Install and configure HAproxy:

```
apt-get update && apt-get install -y haproxy

cat > /etc/haproxy/haproxy.cfg <<EOF
defaults
    maxconn 20000
    mode    tcp
    option  dontlognull
    timeout http-request 10s
    timeout queue        1m
    timeout connect      10s
    timeout client       86400s
    timeout server       86400s
    timeout tunnel       86400s
frontend k8s-master
    bind 192.168.0.10:6443
    mode tcp
    default_backend k8s-control-plane
backend k8s-control-plane
    option tcp-check
    mode tcp
    balance roundrobin
    server master1 192.168.0.11:6443 check fall 3 rise 2
    server master2 192.168.0.12:6443 check fall 3 rise 2
    server master3 192.168.0.13:6443 check fall 3 rise 2
EOF

systemctl restart haproxy && systemctl status haproxy

nc -w 5 -vz 192.168.0.10 6443
```

## Kubernetes nodes preparation

The following commands should be done on each node, better to use tmux.

- Disable swap:

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```

- Configure persistent loading of modules:

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

- Load at runtime:

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

- Setup required sysctl params, these persist across reboots:

```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

- Apply sysctl params without reboot:

```
sudo sysctl --system
```

- Install required packages:

```
sudo apt-get update && \
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

- Add Docker repo:

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

- Install containerd:

```
sudo apt update && \
sudo apt install -y containerd.io
```

- Configure containerd and start service:

```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

- Restart containerd:

```
sudo systemctl enable containerd
sudo systemctl restart containerd
```

- Add the Kubernetes apt repository:

```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- Install K8s packages:

```
sudo apt-get update && \
sudo apt-get install -y kubelet=1.21.1-00 kubeadm=1.21.1-00 kubectl=1.21.1-00

sudo apt-mark hold kubelet kubeadm kubectl
kubectl version --client && kubeadm version

sudo systemctl enable kubelet
```

## Initialize 1st master node

- Create kubeadm config file:

```
cat > kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - "192.168.0.10"
networking:
  podSubnet: 10.112.0.0/16
controlPlaneEndpoint: "192.168.0.10:6443"
EOF
```

- Init node:

```
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
```

- Configure kubectl:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl cluster-info
```

- Install calico network plugin and wait until node becomes Ready:

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
kubectl get pods -n kube-system
kubectl get nodes
```

## Join 2nd and 3rd master nodes

```
sudo kubeadm join 192.168.0.10:6443 --token n838jx.gmrqmr90jayr9y3j \
        --discovery-token-ca-cert-hash sha256:916524a025e27f744f2bef4fb4ef13c5aae87ee2b6b078b17735705234e8554d \
        --control-plane --certificate-key 01b7b2aa26b579538bd9b39db005a4e978691f510b0a315fae60583362d7cddd
```

## Add worker nodes

- WA for containerd:

```
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
```

- Join worker nodes:

```
sudo kubeadm join 192.168.0.10:6443 --token n838jx.gmrqmr90jayr9y3j \
        --discovery-token-ca-cert-hash sha256:916524a025e27f744f2bef4fb4ef13c5aae87ee2b6b078b17735705234e8554d
```


## Simple check node with nginx POD

```
kubectl create -f https://k8s.io/examples/application/deployment.yaml
```
