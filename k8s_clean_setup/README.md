This project is providing a guidance on how to quickly deploy kubernetes cluster for your own non-production needs: demo or lab env.

prerequisites:
- vagrant

To install vagrant, please visit 
https://learn.hashicorp.com/tutorials/vagrant/getting-started-install?in=vagrant/getting-started


Quick main steps:

Prepared Vagrant file
----------------------

```
Vagrant.configure("2") do |config|
    config.vm.provision "shell", inline: <<-SHELL
        apt-get update -y
        echo "10.0.0.10  master-node" >> /etc/hosts
        echo "10.0.0.11  worker-node01" >> /etc/hosts
        echo "10.0.0.12  worker-node02" >> /etc/hosts
    SHELL

    config.vm.define "master" do |master|
      master.vm.box = "bento/ubuntu-18.04"
      master.vm.hostname = "master-node"
      master.vm.network "private_network", ip: "10.0.0.10"
      master.vm.provider "virtualbox" do |vb|
          vb.memory = 4048
          vb.cpus = 2
      end
    end

    (1..2).each do |i|

    config.vm.define "node0#{i}" do |node|
      node.vm.box = "bento/ubuntu-18.04"
      node.vm.hostname = "worker-node0#{i}"
      node.vm.network "private_network", ip: "10.0.0.1#{i}"
      node.vm.provider "virtualbox" do |vb|
          vb.memory = 2048
          vb.cpus = 1
      end
    end

    end
  end
```


Run Virtual Machines
---------------------

just run the vagrant command from inside the folder. Be sure you have vagrant from Hashicorp installed

```
vagrant up
```


Install Docker Container Runtime On All The Nodes
--------------------------------------------------
```
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
Install the required packages for Docker.
```
sudo apt-get update -y
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
Add the Docker GPG key and apt repository.
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install the Docker community edition.

```
sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

Add the docker daemon configurations to use systemd as the cgroup driver.

```
cat <<EOF | sudo tee /etc/docker/daemon.json
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

Start and enable the docker service.

```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Install Kubeadm & Kubelet & Kubectl on all Nodes
--------------------------------------------------

Install the required dependencies.

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

Add the GPG key and apt repository.

```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update apt and install kubelet, kubeadm and kubectl.

```
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
```

Add hold to the packages to prevent upgrades.

```
sudo apt-mark hold kubelet kubeadm kubectl
```

Initialize Kubeadm On Master Node To Setup Control Plane
---------------------------------------------------------

First, set two environment variables. Replace 10.0.0.10 with the IP of your master node.

```
IPADDR="10.0.0.10"
NODENAME=$(hostname -s)
```

Now, initialize the master node control plane configurations using the following kubeadm command.

```
sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=192.168.0.0/16 --node-name $NODENAME --ignore-preflight-errors Swap
```

Use the following commands from the output to create the kubeconfig in master so that you can use kubectl to interact with cluster API.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now, verify the kubeconfig by executing the following kubectl command to list all the pods in the kube-system namespace.

```
kubectl get po -n kube-system
```

Install Calico Network Plugin for Pod Networking
-------------------------------------------------

Execute the following command to install the calico network plugin on the cluster.

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Join Worker Nodes To Kubernetes Master Node
--------------------------------------------

```
kubeadm token create --print-join-command
```

Setup Kubernetes Metrics Server
--------------------------------

To install the metrics server, execute the following metric server manifest file. It deploys metrics server version v0.4.4

```
kubectl apply -f https://raw.githubusercontent.com/scriptcamp/kubeadm-scripts/main/manifests/metrics-server.yaml
```

now this should work

```
kubectl top nodes
```

***_NOTE_*** If you build multi-master-nodes / redundant control plane cluster, you need to use a high available yaml version, e.g. `https://github.com/kubernetes-sigs/metrics-server/releases/download/metrics-server-helm-chart-3.8.2/high-availability.yaml`
