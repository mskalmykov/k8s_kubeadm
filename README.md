1. Update and set up packages:
```bash
sudo apt update && sudo apt upgrade -y && sudo apt install -y vim
```

2. Set up the timezone:
```bash
sudo dpkg-reconfigure tzdata
```

3. Set up vim:
```bash
cat > ~/.vimrc <<EOF
source \$VIMRUNTIME/defaults.vim
set mouse-=a
EOF
sudo cp .vimrc ~root/.vimrc
```

4. Add node names to hosts:
```bash
cat <<EOF | sudo tee -a /etc/hosts

10.240.0.10 cluster1.k8s.my cluster1
10.240.0.11 node1.k8s.my node1
10.240.0.12 node2.k8s.my node2
10.240.0.13 node3.k8s.my node3
EOF
```

5. Distribute node1 ssh keys to other nodes:
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

6. Install keepalived. 
```bash
sudo apt install -y keepalived
```

7. Configure and start keepalived:
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

8. Load required network modules:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe -a overlay br_netfilter
```

9. Add settings to allow iptables to view bridged traffic and to enable routing:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

10. Install Containerd:
First take a look at [https://github.com/containerd/containerd/releases](https://github.com/containerd/containerd/releases) to find the latest version, then change the commands below if needed.
```bash
wget https://github.com/containerd/containerd/releases/download/v1.6.3/containerd-1.6.3-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.6.3-linux-amd64.tar.gz
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

11. Install crictl
Take a look at [https://github.com/kubernetes-sigs/cri-tools/releases/](https://github.com/kubernetes-sigs/cri-tools/releases/) and note the latest version:
```bash
CRICTL_VERSION=v1.22.0
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz" | sudo tar -C /usr/local/bin -xz
```

12. Install kubeadm, kubelet, kubectl

Note: look at https://www.downloadkubernetes.com/ to choose desired version.
```bash
RELEASE="v1.22.9"
ARCH="amd64"
DOWNLOAD_DIR="/usr/local/bin"
cd ${DOWNLOAD_DIR}
sudo curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet,kubectl}
sudo chmod +x {kubeadm,kubelet,kubectl}
```

```bash
RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

sudo systemctl enable --now kubelet
```

13. Install additional packages for kubeadm:
```bash
sudo apt-get install -y conntrack ethtool socat
```

14. Init the first node:
```bash
sudo kubeadm init \
   --control-plane-endpoint=cluster1.k8s.my \
   --apiserver-advertise-address=10.240.0.11 \
   --pod-network-cidr=10.244.0.0/16 \
   --cri-socket=/run/cri-dockerd.sock \
   --upload-certs
```
Save the output from init command to join other nodes!!!

15. Configure kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

16. Install flannel CNI plugin:
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

17. Join node2 and node3:
```bash
sudo kubeadm join cluster1.k8s.my:6443 --token lx6gvi.fkmbi3wpl27h3zk8 \
   --discovery-token-ca-cert-hash sha256:8a08d81a07b740e8c266da30576f4097705863987cbf557ee94f774903fcf4b1 \
   --control-plane \
   --certificate-key 9a277cddf35c2ea28cfb8171a6d3ec47803cac10b4131fb534a5a34181da8891 \
   --apiserver-advertise-address=10.240.0.12 \
   --cri-socket=/run/cri-dockerd.sock
```

```bash
sudo kubeadm join cluster1.k8s.my:6443 --token lx6gvi.fkmbi3wpl27h3zk8 \
   --discovery-token-ca-cert-hash sha256:8a08d81a07b740e8c266da30576f4097705863987cbf557ee94f774903fcf4b1 \
   --control-plane \
   --certificate-key 9a277cddf35c2ea28cfb8171a6d3ec47803cac10b4131fb534a5a34181da8891 \
   --apiserver-advertise-address=10.240.0.13 \
   --cri-socket=/run/cri-dockerd.sock
```

18. Remove taint to enable pod schedulling on the control plane nodes:
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

19. Enable kubectl completion for bash:
```bash
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> .bashrc
```

