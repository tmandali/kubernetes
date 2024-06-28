# Install Kubernetes on Ubuntu
**Prerequisites**

Kubernetes'i Ubuntu makinenize yüklemek için aşağıdaki gereksinimleri karşıladığından emin olun:
- 2 CPUs
- At least 2GB of RAM
- At least 2 GB of Disk Space
- A reliable internet connection

**Step 1: Disable swap**

Kubernetes, mevcut kaynakların anlaşılmasına dayalı olarak çalışmayı planlar. İş yükleri takas kullanmaya başlarsa Kubernetes'in doğru planlama kararları alması zorlaşabilir. Bu nedenle Kubernetes'i kurmadan önce takasın devre dışı bırakılması önerilir.
``` bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Reboot
sudo reboot
free -h
```
**Step 2: Set up hostnames**

Kubernetes kümesiyle çalıştığımızda, Kubernetes'in düğümleri tanımlamak için bu adları kullanabilmesi için düğümlere benzersiz ana makine adları vermemiz gerekir.

``` bash
sudo hostnamectl set-hostname "k8s-master"
# Refresh
exec bash
```
**Step 3: Update the /etc/hosts File for Hostname Resolution**

Ana bilgisayar adlarını ayarlamak yeterli değildir. Ana bilgisayar adlarını IP adresleriyle de eşleştirmemiz gerekiyor. Tüm düğümlerin (veya en azından ana düğümün) /etc/hosts dosyasını aşağıda gösterildiği gibi güncellemelisiniz. 
``` bash
sudo vim /etc/hosts
# Add
192.168.1.117 k8s-master.localdev.me k8s-master  
```

**Step 4: Set up the IPV4 bridge on all nodes**

IPV4 köprüsünü tüm düğümlerde yapılandırmak için her düğümde aşağıdaki komutları yürütün.
``` bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

**Step 5: Install kubelet, kubeadm, and kubectl on each node**

Kubernetes kümesi oluşturmak için her düğüme kubelet, kubeadm ve kubectl yükleyelim. Kubernetes kümesinin yönetilmesinde önemli bir rol oynarlar.

Kubelet, her düğümde çalışan düğüm aracısıdır ve kapların Pod'un spesifikasyonlarında belirtildiği şekilde bir Pod'da çalışmasını sağlamaktan sorumludur. (Pod'lar bir Kubernetes kümesindeki konuşlandırılabilir en küçük birimlerdir).

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

``` bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**Step 6: Install Docker/Containerd**

Docker platformu, konteynerler içinde uygulamalar oluşturmanıza, dağıtmanıza ve çalıştırmanıza olanak tanır. Bu konteynerler hafif ve taşınabilir bir ortam sunarak çeşitli kurulumlarda tutarlı performans sağlar. Docker, konteyner çalışma zamanı olarak görev yapar ve konteynerli uygulamaların verimli yönetimini ve dağıtımını kolaylaştırarak Kubernetes'te hayati bir rol oynar.

https://docs.docker.com/engine/install/ubuntu/

``` bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd.io

# Then, create a default configuration file for containerd and save it as config.toml using the
sudo sh -c "containerd config default > /etc/containerd/config.toml"

# After running these commands, you need to modify the config.toml file to locate the entry that sets "SystemdCgroup" to false and changes its value to true. This is important because Kubernetes requires all its components, and the container runtime uses systemd for cgroups.

sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd.service
sudo systemctl restart kubelet.service

sudo systemctl enable kubelet.service
```

**Step 7: Initialize the Kubernetes cluster on the master node**

Kubeadm kullanarak bir Kubernetes kontrol düzlemini başlattığınızda, kümeyi yönetmek ve düzenlemek için çeşitli bileşenler dağıtılır. Bu bileşenlerin bazı örnekleri kube-apserver, kube-controller-manager, kube-scheduler, vb., kube-proxy'dir. 

``` bash
sudo kubeadm config images pull

# Next, initialize your master node. The --pod-network-cidr flag is setting the IP address range for the pod network.

sudo kubeadm init --control-plane-endpoint="k8s-master.localdev.me" --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=192.168.1.117

# To manage the cluster, you should configure kubectl on the master node. Create the .kube directory in your home folder and copy the cluster's admin configuration to your personal .kube directory. Next, change the ownership of the copied configuration file to give the user the permission to use the configuration file to interact with the cluster. 

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
**Step 9: Tools**

K9S managent tool instalation on the master node

https://webinstall.dev/k9s/

``` bash
curl -sS https://webi.sh/k9s | sh
```

Copy remote kube context
``` bash
scp -r tmr@192.168.1.117:/home/tmr/.kube .
cp -r .kube $HOME/
kubectl get nodes
```