# Introduction
This post describes my experience standing up a Kubernetes cluster using kubeadm, vagrant, and virtualbox.  I use Ubuntu for my target OS.  Once you stand up the master and join a worker node the process pretty much repeats itself to add as many nodes as desired.  All the nodes will be created from the ubuntu/xenial vagrant box.  If you just want a single node cluster to play around with, the last section describes untainting the master.

## Master Node
For this example I only create a single master node.  In a real world production example you would want multinode master for HA.  Follow the steps below to instantiate the vagrant box.
### Configure the Vagrant Box
Create a directory and switch to that directory  
```console
mkdir node01 && cd node01
```

Initialize the Vagrantfile  
Use the Ubuntu/xenial64 box for the VM
```console
vagrant init ubuntu/xenial64
```

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
ping 192.168.33.11
```
SSH into the box.
```console
vagrant ssh
```
Change to root
```console
sudo su
```

In the /etc/hosts file, change the DNS resolution for the hostname to the private network IP
```console
vim /etc/hosts
```
```console
127.0.0.1 localhost
192.168.33.11 kube-master
```

### Add Pre-requisits
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

### Add the Kubernetes package
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

### Initialize Kubernetes
Now it is time to actually initialize Kubernetes on the node.  It seems to work best on the virtualbox VMs if you specify the apiserver advertised address so I included that and since we are going to use flannel as the overlay I included the pod network cidr as well.  Also, a useful command to reset or start over is 'kubeadm reset'.
```console
kubeadm init --apiserver-advertise-address=192.168.33.11 --pod-network-cidr=10.244.0.0/16
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

  kubeadm join 192.168.33.11:6443 --token bgud8e.74zudt1dpo73w0ii --discovery-token-ca-cert-hash sha256:544fa1ba1a1288a14efeb286da52cf0237598f20d1e04a552beee7a691d67eb1
```

The first item configures your conf file for the local kubectl to access the cluster.
```console
mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Applying a Pod Network
The second item in the output above is the pod network you want to use.  There are several you can choose from; Calico, Flannel, Weave, etc.  For this example we will use Flannel.
```console
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

### Untainting the Master Node
If you want to stop here and just use a single node cluster to play around with you can.  All you need to do is untaint the node with the command below.  This will allow you to deploy applications on the master node.  Generally master nodes are tainted so you can't install application workloads.
```console
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Worker Node
This is a simple Kubernetes worker node.  The steps for the base OS are pretty much the same as the master node with a different IP and hostname.  In a real world situation there would be many nodes in a cluster.  

### Configure the Vagrant Box
Create a directory and switch to that directory  
```console
mkdir node02 && cd node02
```

Initialize the Vagrantfile  
Use the Ubuntu/xenial64 box for the VM
```console
vagrant init ubuntu/xenial64
```

Setup the hostname and a static IP  for the node.  Open the Vagrantfile with your favorite text ediror of choice and make the below modifications.
```console
config.vm.box = "ubuntu/xenial64"
  config.vm.hostname = "kube-worker01" # or whatever you want
```
```console
# Create a private network, which allows host-only access to the machine
  # using a specific IP.
   config.vm.network "private_network", ip: "192.168.33.12" # or whatever IP you want to use
```
Now vagrant up the box and make sure you can ping it.
```console
vagrant up
```
```console
ping 192.168.33.12
```
SSH into the box.
```console
vagrant ssh
```
Change to root
```console
sudo su
```

Add DNS entries in the /etc/hosts file for the master and the worker private IP.
```console
vim /etc/hosts
```
```console
127.0.0.1 localhost
192.168.33.11 kube-master
192.168.33.12 kube-worker01
```

### Add Pre-requisits
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

### Add the Kubernetes package
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

### Join the Master
Now we take the kubeadm join command from the result of the kubeadm init and see if we can join the master.  Obviously your token and hash will be different than mine.
```console
  kubeadm join 192.168.33.11:6443 --token bgud8e.74zudt1dpo73w0ii --discovery-token-ca-cert-hash sha256:544fa1ba1a1288a14efeb286da52cf0237598f20d1e04a552beee7a691d67eb1
```
The result should look like the following.
```console
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

### Verify the Node Joined
Now we want to switch back to the master node.  There are a couple things I always check.  It might take a few minutes to sync-up, so give it a minute.
```console
kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
kube-master     Ready    master   32m   v1.12.2
kube-worker01   Ready    <none>   26m   v1.12.2
```

```console
kubectl get pods -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
coredns-576cbf47c7-fswt5              1/1     Running   0          31m
coredns-576cbf47c7-vcjwv              1/1     Running   0          31m
etcd-kube-master                      1/1     Running   0          31m
kube-apiserver-kube-master            1/1     Running   0          30m
kube-controller-manager-kube-master   1/1     Running   0          30m
kube-flannel-ds-amd64-k7zn9           1/1     Running   0          26m
kube-flannel-ds-amd64-njm79           1/1     Running   0          29m
kube-proxy-lk4sb                      1/1     Running   0          31m
kube-proxy-lnmw2                      1/1     Running   0          26m
kube-scheduler-kube-master            1/1     Running   0          30m
```

## Conclusion
This should give you a Kubernetes environment to play around.  Just build another worker node and add it if you want more nodes.  Try deploy an application with the below command.
```console
kubectl run --image=nginx app
```

