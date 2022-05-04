### Update and set up packages:
```bash
sudo apt update && sudo apt upgrade -y && sudo apt install -y vim curl
```

### Set up the timezone:
```bash
sudo dpkg-reconfigure tzdata
```

### Set up vim:
```bash
cat > ~/.vimrc <<EOF
source \$VIMRUNTIME/defaults.vim
set mouse-=a
EOF
sudo cp .vimrc ~root/.vimrc
```

### Add node names to hosts:
```bash
cat <<EOF | sudo tee -a /etc/hosts

10.240.0.10 cluster1.k8s.my cluster1
10.240.0.11 node1.k8s.my node1
10.240.0.12 node2.k8s.my node2
10.240.0.13 node3.k8s.my node3
EOF
```

### Distribute node1 ssh keys to other nodes:
```bash
node1$ vi .ssh/id_rsa
node1$ chmod 0600 .ssh/id_rsa
node1$ cat .ssh/authorized_keys
```

```bash
node2-3$ cat >> .ssh/authorized_keys <<EOF
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDMgJW7aVveniLwqtLZ2+mGjlW5SxcsLcoAGraPhh1lg1gepXxeu8y+XR9zZqXIeqQINFvNejnP48L7n24NmpDsj3cf2BH/vMIIsphhSP+rQMWnalkRaPYk8nQMv2ZtXXOxWrz6jJw96RezCVmERE1YPe5C3+HY22LB+iTLjfE6nCgCXr+AiXGn+0EMJwyMSrCAjzYL5tkRuzZGEDPvrzwpsXaTGheIePqxlrpp3pElCQAmHO8DdLwqSKSGYGTq3whR1WDFxVjPbCluS1aV4hnNpg4ZE4SMKLUssh+42JWXeSnrzNKxnTlKMrI2KpGKU8a3YmgOcZdv5es8ouaEY6n9
EOF
```

### Install keepalived. 
```bash
sudo apt install -y keepalived
```

### Configure and start keepalived:
node1:

```bash
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface eth1
    virtual_router_id 17
    priority 100
    advert_int 1
    virtual_ipaddress {
        10.240.0.10
    }
}
EOF
sudo systemctl start keepalived
```

node2:

```bash
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 17
    priority 90
    advert_int 1
    virtual_ipaddress {
        10.240.0.10
    }
}
EOF
sudo systemctl start keepalived
```

node3:

```bash
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 17
    priority 80
    advert_int 1
    virtual_ipaddress {
        10.240.0.10
    }
}
EOF
sudo systemctl start keepalived
```

### Load required network modules:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe -a overlay br_netfilter
```

### Add settings to allow iptables to view bridged traffic and to enable routing:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

### Install Containerd:

First take a look at [https://github.com/containerd/containerd/releases](https://github.com/containerd/containerd/releases) to find the latest version, then change the commands below if needed.
```bash
wget https://github.com/containerd/containerd/releases/download/v1.6.4/containerd-1.6.4-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.6.4-linux-amd64.tar.gz
sudo wget -O /usr/lib/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -e 's/SystemdCgroup = false/SystemdCgroup = true/' -i /etc/containerd/config.toml
``` 
Download and install runc (note the latest version):
```bash
sudo wget -O /usr/local/sbin/runc \
       https://github.com/opencontainers/runc/releases/download/v1.1.1/runc.amd64
sudo chmod 0755 /usr/local/sbin/runc
```
Download and install CNI plugins from [https://github.com/containernetworking/plugins/releases](https://github.com/containernetworking/plugins/releases):
```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```
Enable and run containerd:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### Install kubeadm, kubelet, kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Init the first node:
```bash
sudo kubeadm init \
   --control-plane-endpoint=cluster1.k8s.my \
   --apiserver-advertise-address=10.240.0.11 \
   --pod-network-cidr=10.244.0.0/16 \
   --upload-certs
```
Save the output from init command to join other nodes!!!

### Configure kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install flannel CNI plugin (first node only):
```bash
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
vi kube-flannel.yml # Add - --iface=eth1 to container args
kubectl apply -f kube-flannel.yml
```

### Join node2 and node3:
```bash
sudo kubeadm join cluster1.k8s.my:6443 \
    --token sjkleg.r53wu4cfntnc3a0y \
    --discovery-token-ca-cert-hash sha256:d306d50bec853fcb24f7535ea5226abcf200163ab7825113b5a67fc1ff7ab8ba \
    --certificate-key 7682de9c2fd19f8a4bde2f2072537084736a05bfd66b6936f46cb792602ba803 \
    --control-plane \
    --apiserver-advertise-address=10.240.0.12
```

```bash
sudo kubeadm join cluster1.k8s.my:6443 \
    --token sjkleg.r53wu4cfntnc3a0y \
    --discovery-token-ca-cert-hash sha256:d306d50bec853fcb24f7535ea5226abcf200163ab7825113b5a67fc1ff7ab8ba \
    --certificate-key 7682de9c2fd19f8a4bde2f2072537084736a05bfd66b6936f46cb792602ba803 \
    --control-plane \
    --apiserver-advertise-address=10.240.0.13
```

### Remove taint to enable pod schedulling on the control plane nodes:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master- 
```

### Enable kubectl completion for bash:
```bash
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```
