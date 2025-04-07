Kubernetes Installation
Step 1

-------->>>> sudo  nano  /etc/hosts

Add the IP Address entries. This will allow for hostname resolution. Replace the IP addresses and hostnames with your own.
  For Kubernets Master
-------->>>> sudo hostnamectl set-hostname "k8s-master-node"
For Kubernetes Worker
-------->>>> sudo hostnamectl set-hostname "k8s-worker-node"
After this uses -------->>>>  sudo exec bash
-------->>>> Verify- hostname

Step 2: Disable swap space (all nodes)
-------->>>> sudo swapoff -a
To check whether the swap space has been disabled, run the command: 
------->>>> swapon --show

Step 3: Load Containerd modules (all nodes)

Containerd is the standard container runtime for Kubernetes. For the seamless running of this runtime, it is recommended 
that the overlay and br_netfilter kernel modules be loaded. To enable and load these modules, run the following commands:

-------->>> sudo modprobe overlay
-------->>> sudo modprobe br_netfilter

Create a configuration file as shown and specify the modules to load them permanently.

[sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF]

Step 4: 
Configure Kubernetes IPv4 networking (all nodes)
It's important to configure Kubernetes networking so that pods can communicate with each other and outside environments without a hitch.

Create a Kubernetes configuration file in the /etc/sysctl.d/ directory.
sudo nano   /etc/sysctl.d/k8s.conf

Add the following lines:
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward   = 1

Save and exit. Then apply the settings by running the following command:
sudo sysctl --system

Step 5: 
Install Docker (on all nodes)
In Kubernetes, Docker creates and manages containers on each node in the cluster while Kubernetes orchestrates the deployment, scaling, 
monitoring, and management of containerized applications. The two work in tandem to manage microservice applications at a large scale.

Ubuntu provides Docker from its default repositories. To install it, run the following commands on all nodes.

-------->>> sudo apt update
-------->>> sudo apt install docker.io -y

-------->>> sudo systemctl status docker

In addition, be sure to enable the Docker daemon to autostart on system startup or upon a reboot.

-------->>> sudo systemctl enable docker

The installation of Docker also comes with containerd, a lightweight container runtime that streamlines the running and management of 
containers. As such, configure containerd to ensure it runs reliably in the Kubernetes cluster. First, create a separate directory

-------->>> sudo mkdir /etc/containerd

Next, create a default configuration file for containerd.
-------->>> sudo sh -c "containerd config default > /etc/containerd/config.toml"

Update the SystemdCgroup directive by setting it to true
-------->>> sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml

Next, restart the containerd service to apply the changes made.
-------->>> sudo systemctl restart containerd.service

verify that the containerd service is running as expected.
-------->>> sudo systemctl status containerd.service

Step 6: Install Kubernetes components (on all nodes)
The next step is to install Kubernetes. Be sure to install the prerequisite packages on all nodes.

-------->>> sudo apt-get install curl ca-certificates apt-transport-https  -y

Add the Kubernetes GPG signing key.
-------->>> curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

Add the official Kubernetes repository to your system.
-------->>> echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

Once added, update the sources list for the system to recognize the newly added repository.
-------->>> sudo apt update

To install Kubernetes, we will install three packages:

kubeadm: This is a command-line utility for setting up Kubernetes clusters. It automates setting up a cluster, streamlining container deployments, and 
abstracting any complexities within the cluster. With kubeadm you can initialize the control-plane, configure networking, and join a remote node to a cluster.

kubelet: This is a component that actively runs on each node in a cluster to oversee container management. It takes instructions from the master node 
and ensures containers run as expected.

kubectl: This is a CLI tool for managing various cluster components including pods, nodes, and the cluster. You can use it to deploy applications and inspect, 
monitor, and view logs.

To install these salient Kubernetes components, run the following command:
-------->>> sudo apt install kubelet kubeadm kubectl -y

Step 7: Initialize Kubernetes cluster (on master node)
The next step is to initialize the Kubernetes cluster on the master node. This configures the master node as the control plane. To initialize the cluster, run the command 
shown. The --pod-network-cidr indicates a unique pod network for the cluster, in this case, the 10.10.0.0 network with a CIDR of /16.

-------->>> sudo kubeadm init --pod-network-cidr=10.10.0.0/16

Therefore, run the commands as a regular user. Create a .kube directory in your home directory.

-------->>> mkdir -p $HOME/.kube

Next, copy the cluster's configuration file to the .kube directory.

-------->>> sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

Lastly, configure the ownership of the copied configuration file to allow the user to leverage the config file to manage the cluster.

-------->>> sudo chown $(id -u):$(id -g) $HOME/.kube/config

Step 8: Install Calico network add-on plugin

Developed by Tigera, Calico provides a network security solution in a Kubernetes cluster. It secures communication between individual pods as well 
as pods and external services. By auto-assigning IP addresses to pods, it ensures smooth communication between them.

With this in mind, deploy the Calico operator using the kubectl CLI tool.

-------->>> kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

Next, download Calico’s custom-resources file using the curl command.

-------->>> curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O

Update the CIDR defined in the custom-resources file to match the pod's network.

-------->>> sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.10.0.0\/16/g' custom-resources.yaml

Then create resources defined in the YAML file.

-------->>> kubectl create -f custom-resources.yaml

At this point, the master node, which acts as the control plane, is the only node that is available in the cluster. To verify this, run the command:

-------->>> kubectl get nodes

Step 9: Add worker nodes to the cluster (on worker nodes)
With the master node configured, the remaining step is to add the worker nodes to the cluster. To do this, run the kubeadm join command generated in Step 7 
when initializing the cluster.

So head back to the worker nodes and run the kubeadm join command below in each node.

-------->>> kubeadm join 84.32.214.62:6443 --token 2i5iru.f4m1vbxyc9w2ve7q \
        --discovery-token-ca-cert-hash sha256:29edadbccfc479ca35d1480058069dcadf7f694e510756fd153e4f65ae7f39f8

Now check the nodes in the cluster once again.
-------->>> kubectl get pods -A


