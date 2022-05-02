1. Update and set up packages:
    sudo apt update && sudo apt upgrade -y && sudo apt install -y vim

2. Set up the timezone:
    sudo dpkg-reconfigure tzdata

3. Set up vim:
cat > ~/.vimrc <<EOF
source \$VIMRUNTIME/defaults.vim
set mouse-=a
EOF
sudo cp .vimrc ~root/.vimrc

4. Add node names to hosts:
cat <<EOF | sudo tee -a /etc/hosts

10.240.0.10 cluster1.k8s.my cluster1
10.240.0.11 node1.k8s.my node1
10.240.0.12 node2.k8s.my node2
10.240.0.13 node3.k8s.my node3
EOF

5. Distribute node1 ssh keys to other nodes:
node1$ vi .ssh/id_rsa
node1$ chmod 0600 .ssh/id_rsa
node1$ cat .ssh/authorized_keys

node2-3$ cat >> .ssh/authorized_keys <<EOF
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDMgJW7aVveniLwqtLZ2+mGjlW5SxcsLcoAGraPhh1lg1gepXxeu8y+XR9zZqXIeqQINFvNejnP48L7n24NmpDsj3cf2BH/vMIIsphhSP+rQMWnalkRaPYk8nQMv2ZtXXOxWrz6jJw96RezCVmERE1YPe5C3+HY22LB+iTLjfE6nCgCXr+AiXGn+0EMJwyMSrCAjzYL5tkRuzZGEDPvrzwpsXaTGheIePqxlrpp3pElCQAmHO8DdLwqSKSGYGTq3whR1WDFxVjPbCluS1aV4hnNpg4ZE4SMKLUssh+42JWXeSnrzNKxnTlKMrI2KpGKU8a3YmgOcZdv5es8ouaEY6n9
EOF

6. Install keepalived. 
sudo apt install -y keepalived

7. Configure and start keepalived:
node1:

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

node2:

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

node3:

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


8. Load required network modules:
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe -a overlay br_netfilter

9. Add settings to allow iptables to view bridged traffic and to enable routing:
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

10. Install Docker Engine:
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker 

11. Install cri-dockerd
export VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4)
echo $VER
wget https://github.com/Mirantis/cri-dockerd/releases/download/${VER}/cri-dockerd-${VER}-linux-amd64.tar.gz
tar xvf cri-dockerd-${VER}-linux-amd64.tar.gz cri-dockerd
sudo mv cri-dockerd /usr/local/bin/
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket

#12. Install CNI plugins (note the latest version):
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

#13. Configure CNI bridge and loopback
sudo mkdir -p /etc/cni/net.d/
POD_CIDR=10.200.3.0/24   # change third octet to node number
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF

cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF

14. Install crictl
CRICTL_VERSION=v1.22.0
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz" | sudo tar -C /usr/local/bin -xz

15. Install kubeadm, kubelet, kubectl

# Look at https://www.downloadkubernetes.com/ to choose version:
RELEASE="v1.22.9"
ARCH="amd64"
DOWNLOAD_DIR="/usr/local/bin"
cd ${DOWNLOAD_DIR}
sudo curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet,kubectl}
sudo chmod +x {kubeadm,kubelet,kubectl}

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

sudo systemctl enable --now kubelet

16. Install additional packages for kubeadm:
sudo apt-get install -y conntrack ethtool socat

17. Init the first node:
sudo kubeadm init \
   --control-plane-endpoint=cluster1.k8s.my \
   --apiserver-advertise-address=10.240.0.11 \
   --pod-network-cidr=10.244.0.0/16 \
   --cri-socket=/run/cri-dockerd.sock \
   --upload-certs
Save the output from init command to join other nodes!!!

18. Configure kubectl:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

19. Install flannel CNI plugin:
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

19. Join node2 and node3:
sudo kubeadm join cluster1.k8s.my:6443 --token lx6gvi.fkmbi3wpl27h3zk8 \
   --discovery-token-ca-cert-hash sha256:8a08d81a07b740e8c266da30576f4097705863987cbf557ee94f774903fcf4b1 \
   --control-plane \
   --certificate-key 9a277cddf35c2ea28cfb8171a6d3ec47803cac10b4131fb534a5a34181da8891 \
   --apiserver-advertise-address=10.240.0.12 \
   --cri-socket=/run/cri-dockerd.sock

sudo kubeadm join cluster1.k8s.my:6443 --token lx6gvi.fkmbi3wpl27h3zk8 \
   --discovery-token-ca-cert-hash sha256:8a08d81a07b740e8c266da30576f4097705863987cbf557ee94f774903fcf4b1 \
   --control-plane \
   --certificate-key 9a277cddf35c2ea28cfb8171a6d3ec47803cac10b4131fb534a5a34181da8891 \
   --apiserver-advertise-address=10.240.0.13 \
   --cri-socket=/run/cri-dockerd.sock

20. Remove taint to enable pod schedulling on the control plane nodes:
kubectl taint nodes --all node-role.kubernetes.io/master-

21. Enable kubectl completion for bash:
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> .bashrc

