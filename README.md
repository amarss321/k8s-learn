# k8s-learn


# Building a Kubernetes Cluster
# Lesson URL: https://acloudguru-content-attachment-production.s3-accelerate.amazonaws.com/1689779321464-1030%20-%20Building%20a%20Kubernetes%20Cluster%20Lesson%20Reference.txt

# Relevant Documentation

- Installing kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Creating a cluster with kubeadm: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

# Lesson Reference

# If you are using Cloud Playground, create three servers with the following settings:

- Distribution: Ubuntu 20.04 Focal Fossa LTS
- Size: medium

# If you wish, you can set an appropriate hostname for each node.

# On the control plane node:

sudo hostnamectl set-hostname k8s-control

# On the first worker node:

sudo hostnamectl set-hostname k8s-worker1

# On the second worker node:

sudo hostnamectl set-hostname k8s-worker2

# On all nodes, set up the hosts file to enable all the nodes to reach each other using these hostnames:

sudo vi /etc/hosts

# On all nodes, add the following at the end of the file. You will need to supply the actual private IP address for each node:

<control plane node private IP> k8s-control
<worker node 1 private IP> k8s-worker1
<worker node 2 private IP> k8s-worker2

# Log out of all three servers and log back in to see these changes take effect.

# On all nodes, set up Docker Engine and containerd. You will need to load some kernel modules and modify some system settings as part of this
process:

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup; params persist across reboots:

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot:

sudo sysctl --system

# Set up the Docker Engine repository:

sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https

# Add Dockerâ€™s official GPG key:

sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up the repository:

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update the apt package index:

sudo apt-get update

# Install Docker Engine, containerd, and Docker Compose:

VERSION_STRING=5:23.0.1-1~ubuntu.20.04~focal
sudo apt-get install -y docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin

# Add your 'cloud_user' to the docker group:

sudo usermod -aG docker $USER

# Log out and log back in so that your group membership is re-evaluated.

# Make sure that 'disabled_plugins' is commented out in your config.toml file:

sudo sed -i 's/disabled_plugins/#disabled_plugins/' /etc/containerd/config.toml

# Restart containerd:

sudo systemctl restart containerd

# On all nodes, disable swap:

sudo swapoff -a

# On all nodes, install kubeadm, kubelet, and kubectl:

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update && sudo apt-get install -y kubelet=1.27.0-00 kubeadm=1.27.0-00 kubectl=1.27.0-00

sudo apt-mark hold kubelet kubeadm kubectl

# On the control plane node only, initialize the cluster and set up kubectl access:

sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.27.0

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify the cluster is working:

kubectl get nodes

# Install the Calico network add-on:

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml

# Get the join command (this command is also printed during kubeadm init; feel free to simply copy it from there):

kubeadm token create --print-join-command

# Copy the join command from the control plane node. Run it on each worker node as root (i.e., with sudo ):

sudo kubeadm join ...

# On the control plane node, verify all nodes in your cluster are ready. Note that it may take a few moments for all of the nodes to enter the READY state:

kubectl get nodes
--------------------------------------------------------------------------------------------------------------
#Relevant Documentation
#Namespaces
#Lesson Reference
#List namespaces in the cluster.

kubectl get namespaces

#Specify a namespace when listing other objects such as pods.

kubectl get pods -n kube-system

#Create a namespace.

kubectl create namespace my-namespace




------------------------------------------------------------------------------

https://acloudguru-content-attachment-production.s3-accelerate.amazonaws.com/1693337516325-devops-wb002%20-%20S03-L04%20Safely%20Draining%20a%20K8s%20Node.txt

Lesson Reference

# Begin by creating some objects. We will examine how these objects are affected by the drain process.
# First, create a pod

vi pod.yml

apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
  image: nginx:1.14.2
  ports:
  - containerPort: 80
  restartPolicy: OnFailure


# Run apply command

kubectl apply -f pod.yml

# Create a deployment with two replicas.

vi deployment.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: my-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-deployment
  template:
    metadata:
      labels:
        app: my-deployment
    spec:
      containers:
      - name: nginx
      image: nginx:1.14.2
      ports:
      - containerPort: 80

# Run apply command

kubectl apply -f deployment.yml

# Get a list of pods. You should see the pods you just created (including the two replicas from the deployment). Take note of
# which node these pods are running on.

kubectl get pods -o wide

# Drain the node which the my-pod pod is running.

kubectl drain <node name> --ignore-daemonsets --force

# Check your list of pods again. You should see the deployment replica pods being moved to the remaining node. The regular
# pod will be deleted.

kubectl get pods -o wide

# Uncordon the node to allow new pods to be scheduled there again.

kubectl uncordon <node name>

# Delete the deployment created for this lesson.

kubectl delete deployment my-deployment
