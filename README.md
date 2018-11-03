# Introduction
This post describes my experience standing up a Kubernetes cluster using kubeadm, vagrant, and virtualbox.  I use Ubuntu for my target OS.  Once you stand up the master and join a worker node the process pretty much repeats itself to add as many nodes as desired.
## Create Nodes
All the nodes will be created from the ubuntu/xenial vagrant box.  There are a few preqrequisits that need to also be added before adding the Kubernetes components.

### Master Node
For this example I only create a single master node.  In a real world production example you would want multinode master for HA.  Follow the steps below to instantiate the vagrant box.
1. Create a directory and switch to that directory  
```console
mkdir node01 && cd node01
```

2. Initialize the Ubuntu Vagrantfile  
```console
vagrant init ubuntu/xenial64
```

3. Set hostname and IP
Setup the hostname and a static IP  for the node.  Open the Vagrantfile with your favorite text ediror of choice and make the below modifications.
```console
config.vm.box = "ubuntu/xenial64"
  config.vm.hostname = "kube-master" # or whatever you want
```
```console
# Create a private network, which allows host-only access to the machine
  # using a specific IP.
   config.vm.network "private_network", ip: "192.168.33.11" # or whatever IP you want to use
```
Now vagrant up the box and make sure you can ping it.
```console
vagrant up
```
```console
ping 192.168.33.10
```
SSH into the box.
```console
vagrant ssh
```
Change to root
```console
sudo su
```

Change the DNS resolution for the hostname to the private network IP
```console
192.168.33.10 kube-master
```

4. Add Pre-requisits
Update the packages and add a few support applications (ie... Docker).  Typically, once I establish the required pieces I will add them to the provision section of the Vagrantfile so this step is automated.  , This is a learning experience so good idea to do them each for now.
```console
apt-get update
```
Some required and helper packages
```console
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common 
```
	
Now add Docker Community Edition
```console
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
```
```console
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
```console
apt-get update && apt-get install -y docker-ce
```
Verify docker is installed.
```console
docker ps
```

Add the Kubernetes package
```console
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```
```console
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

Update the package manager and install kubadm
```console
apt-get update && apt-get install -y kubeadm
```

Now initialize the Kubernetes components.
```console
kubeadm init
```

If all goes well you should receive  messages similar to the below.
```console
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.33.10:6443 --token bgud8e.74zudt1dpo73w0ii --discovery-token-ca-cert-hash sha256:544fa1ba1a1288a14efeb286da52cf0237598f20d1e04a552beee7a691d67eb1
```

The first item configures your conf file for the local kubectl to access the cluster.
```console
mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
The second item is the pod network you want to use.  There are several you can choose from; Calico, Flannel, Weave, etc.  For this example we can use Calico.
```console
kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
```

If you want to stop here and just use a single node cluster to play around with you can.  All you need to do is untaint the node with the command below.  This will allow you to deploy applications on the master node.  Generally master nodes are tainted so you can't install application workloads.
```console
kubectl taint nodes --all node-role.kubernetes.io/master-
```

