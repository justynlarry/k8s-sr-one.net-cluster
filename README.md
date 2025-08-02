# Highly Available Kubernetes Homelab
![Docker Build](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)

This repository documents the setup and configuration of a five-node Kubernetes cluster built on Proxmox virtual machines. This project emphasizes high availability for the control plane and leverages key tools like MetalLB for bare-metal load balancing, an NGINX Ingress Controller for traffic routing, and Cloudflare Tunnel for secure external access. The cluster hosts a custom Dockerized web application, showcasing a full end-to-end deployment.

# Features 
Five-Node Cluster: A robust setup with a mix of control plane and worker nodes.
High-Availability (HA) Control Plane: Utilizes kube-vip to ensure the Kubernetes API server remains accessible even if a control plane node fails.
Bare-Metal Load Balancing: MetalLB provides LoadBalancer services, integrating seamlessly with your on-premises network.
NGINX Ingress Controller: Manages external access to services within the cluster, supporting host-based routing.
Secure External Access: Cloudflare Tunnel creates a secure, outbound-only connection to expose your applications without opening inbound firewall ports.
Custom Dockerized Application: Demonstrates how to deploy and manage a simple web application.

# Topology
The cluster consists of five virtual machines hosted on Four Proxmox nodes, with distinct roles:

```
Proxmox Host    VM Name        Role           IP Address (Example)
red-one        red-two         Control-P      192.168.0.101
red-one        red-three       Worker  	      192.168.0.102
secondsun      gold-two	       Control    	  192.168.0.103
hth            blue-two        Control    	  192.168.0.104
deathStar      gray-three      Worker	      192.168.0.105
```

** Control-P (Primary Control Plane): This node initializes the cluster and is responsible for running control plane components.
** Control (Secondary Control Plane): These nodes join the cluster as additional control plane members, contributing to high availability.
** Worker: These nodes run your application workloads. 

# Components
Kubernetes: Container orchestration platform.
Proxmox: Virtualization platform for hosting VMs.
Containerd: Container runtime.
Calico: CNI plugin for network policies and connectivity.
Kube-VIP: Provides a virtual IP (VIP) for the Kubernetes API server for high availability.
MetalLB: A bare-metal load-balancer implementation for Kubernetes, providing external IP addresses for services.
NGINX Ingress Controller: An Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer.
Cloudflare Tunnel: Connects your infrastructure to Cloudflare without opening inbound firewall ports.
Docker: Used for building and pushing application images.

# File Structure
```
.
├── html/                     # Directory containing your web application's HTML files
├── Dockerfile                # Dockerfile for building your web application image
├── metallb-config.yaml       # MetalLB IP address pool and L2 advertisement configuration
├── kube-apiserver-lb.yaml    # MetalLB Service for the Kubernetes API server VIP
├── portfolio-deployment.yaml # Kubernetes Deployment manifest for your web application
├── portfolio-service.yaml    # Kubernetes Service manifest for your web application
├── portfolio-ingress.yaml    # Kubernetes Ingress manifest for routing traffic to your application
└── README.md                 # This README file
```

# Goals Achieved
- High-availability Control Plane (via kube-vip)
- Bare-metal Load Balancer (via MetalLB)
- External Ingress via Cloudflare Tunnel
- GitOps-style deployment manifests for easy management
- Setup and Installation Guide

# This section provides a detailed, step-by-step guide to setting up your Kubernetes homelab cluster.

1. Infrastructure Setup
Before you begin, ensure you have five virtual machines provisioned on your Proxmox nodes.

Preparing the VMs
Operating System: Ubuntu Server 20.04 LTS (or newer stable release).
Resources:
Control Plane Nodes (mandalore, goldnine, chewbacca): 2 CPUs, 4GB RAM each.
Worker Nodes (falcon, shire): 2 CPUs, 2GB RAM each.
Network: All VMs must be on the same virtual network with static IP addresses assigned. Ensure network connectivity between all VMs.
Install Container Runtime
Install a container runtime (e.g., containerd) on all nodes:

```
# Example for containerd (ensure you follow official containerd installation guide for your OS)
sudo apt update
sudo apt install -y containerd.io
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Disable Swap
Disable swap on all nodes. This is a critical prerequisite for Kubernetes.
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
Important: Double-check that the swap line in your /etc/fstab file is commented out (#). If not, swap will be re-enabled on reboot, preventing kubelet from starting.

2. Kubernetes Installation
Install kubeadm, kubelet, and kubectl on all nodes.

```
# Update package lists
sudo apt-get update

# Install necessary packages
sudo apt-get install -y apt-transport-https ca-certificates curl

# Download and dearmor the Kubernetes GPG key to the keyrings directory
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes APT repository for v1.30
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update apt package index with the new repository
sudo apt-get update

# Install kubelet, kubeadm, and kubectl
sudo apt-get install -y kubelet kubeadm kubectl

# Hold these packages to prevent unintended upgrades
sudo apt-mark hold kubelet kubeadm kubectl
```

Increase Max Buffer (For Cloudflare Tunnel Nodes)
If you plan to run Cloudflare Tunnel on certain nodes, increase the max buffer by adding these lines to /etc/sysctl.conf on those specific nodes (e.g., control plane nodes or dedicated Cloudflare Tunnel nodes):

```
sudo sysctl -w net.core.rmem_max=2500000
sudo sysctl -w net.core.wmem_max=2500000
sudo nano /etc/sysctl.conf
# Add the following lines to the end of the file:
# net.ipv4.ping_group_range = 0 2147483647
# net.core.rmem_max=2500000  # Increase max receive buffer to ~2.5MB
# net.core.wmem_max=2500000  # Increase max send buffer to ~2.5MB
After modifying sysctl.conf, apply the changes: sudo sysctl -p.
```

2a. Set Up kube-vip (HA VIP)
kube-vip provides a shared IP address (Virtual IP - VIP) that exposes the Kubernetes API server as a single endpoint, ensuring high availability even if a control plane node goes down. This VIP will float between your control plane nodes.

Choosing your VIP:
Subnet: The VIP must be on the same subnet as your control plane nodes.
Availability: It must not be currently in use by any other machine or VM.
Reservation: It's highly recommended to reserve this IP address outside your DHCP range to prevent conflicts.
Interface: It must be within the same network interface range as your control plane VM interfaces (e.g., ens18).
Example: If your control plane nodes are 192.168.1.101, 192.168.1.103, and 192.168.1.105, your subnet is 192.168.1.0/24. You might choose 192.168.1.241 as your VIP. You can verify its availability with ping <VIP_ADDRESS> or arp -a | grep <VIP_ADDRESS>.

Install kube-vip on ALL Control Nodes (mandalore, goldnine, chewbacca):
Replace 192.168.1.241 with your chosen VIP and ens18 with your network interface name.

```
export VIP=192.168.1.241 # Replace with your chosen VIP
export INTERFACE=ens18   # Replace with your network interface name (e.g., ens18, eth0)
export KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")

sudo ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION
sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip manifest pod --interface $INTERFACE --address $VIP --controlplane --services --arp --leaderElection | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
```

2b. Initialize the Cluster on the Primary Control Plane (red-two)
On your designated primary control plane node (mandalore), initialize the Kubernetes cluster using the VIP:

```
sudo kubeadm init --control-plane-endpoint=$VIP --upload-certs
```
After initialization, copy the kubeconfig to your local user:

```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install a CNI Plugin
On the primary control plane (mandalore), install a CNI plugin (e.g., Calico) to enable network communication between pods:

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

2c. Join Secondary Control Nodes (gold-two, blue-two)
On each of the secondary control nodes, use the kubeadm join command provided in the output of the kubeadm init command (from step 2b) to join them as control plane nodes. The command will look similar to this:

```
sudo kubeadm join 192.168.1.241:6443 --control-plane --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> --certificate-key <key>
```

2d. Join Worker Nodes (gary-three, red-three)
On each of the worker nodes, use the kubeadm join command (also from the output of step 2b) to join them as worker nodes. Omit the --control-plane flag for worker nodes. The command will look similar to this:

```
sudo kubeadm join 192.168.1.241:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Verify Node Status
On any control plane node, verify that all nodes are in the Ready state:
```
kubectl get nodes
```
3. MetalLB Installation
MetalLB provides network load-balancing functionality for your bare-metal Kubernetes cluster.
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```
Create a Secret for Memberlist
```
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
Define an IP Address Pool
Create a file named metallb-config.yaml with an IP address range that MetalLB can use for LoadBalancer services. Ensure this range is outside your DHCP scope and not in use by other devices.

```
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: portfolio-ip-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.246-192.168.1.250 # Adjust this range to your network
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: portfolio-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - portfolio-ip-pool
```
Apply the configuration:

```
kubectl apply -f metallb-config.yaml
```
Configure MetalLB for Kubernetes API Server VIP (Optional but recommended)
This step configures MetalLB to serve the kube-apiserver VIP, providing an additional layer of load balancing and redundancy for API server access.

Create a Service manifest kube-apiserver-lb.yaml:

```
# kube-apiserver-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-apiserver-lb
  namespace: kube-system
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    component: kube-apiserver
  ports:
    - name: https
      port: 6443
      targetPort: 6443
```
Apply the configuration:

```
kubectl apply -f kube-apiserver-lb.yaml
```
4. NGINX Ingress Controller Installation
Deploy the NGINX Ingress Controller to manage external access to your cluster's services.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/baremetal/deploy.yaml
```

Expose Ingress Controller via LoadBalancer
Patch the NGINX Ingress Controller service to be of type LoadBalancer, allowing MetalLB to assign an external IP.

```
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'
```

Verify External IP
Check the external IP assigned by MetalLB to the NGINX Ingress Controller:

```
kubectl get svc -n ingress-nginx
```
Look for the EXTERNAL-IP in the output. This IP will be used by Cloudflare Tunnel.

5. Application Deployment
This section covers building your Docker image, pushing it to a registry, and deploying your application to Kubernetes.

Docker Login and Permissions
Log in to Docker Hub (or your preferred registry) via the CLI:
```
docker login
```
Add your user to the Docker group on the server(s) where you'll be building images:

```
sudo usermod -aG docker ${USER}
```
# You may need to log out and back in for the changes to take effect.
Build and Push Docker Image
Ensure your html/ folder contains your website content.

Create a Dockerfile in the root of your project:

Dockerfile
```
# Dockerfile
FROM nginx:alpine
COPY html/ /usr/share/nginx/html
```
Build your Docker image:

```
docker build -t larryman77/k8s-portfolio:v1.0 .
```
Push your image to Docker Hub:
```
docker push larryman77/k8s-portfolio:v1.0
```
Updating Your Application
If you make changes to your website (content in html/), follow these steps to update your deployed application:

Rebuild the Docker image:
```
docker build -t larryman77/k8s-portfolio:v1.0 .
```
Re-push the Docker image to Docker Hub: This uploads the new image, updating the v1.0 tag to point to your latest build.
```
docker push larryman77/k8s-portfolio:v1.0
```
Force a rollout restart of your Kubernetes Deployment: This tells Kubernetes to pull the new image and update your running pods.
```
kubectl rollout restart deployment/portfolio
```
Create Kubernetes Manifests:
portfolio-deployment.yaml: Defines your application deployment.
```
# portfolio-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portfolio
spec:
  replicas: 4 # Adjust as needed based on your cluster size and desired redundancy
  selector:
    matchLabels:
      app: portfolio
  template:
    metadata:
      labels:
        app: portfolio
    spec:
      containers:
        - name: portfolio
          image: larryman77/k8s-portfolio:v1.0 # Ensure this matches your Docker Hub image
          ports:
            - containerPort: 80
```
portfolio-service.yaml: Defines a Service to expose your deployment within the cluster.

```
# portfolio-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: portfolio-service
spec:
  selector:
    app: portfolio
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
portfolio-ingress.yaml: Defines an Ingress resource to expose your service via the NGINX Ingress Controller. Replace www.sr-one.net with your actual domain.
```
# portfolio-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: portfolio-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: www.sr-one.net # Replace with your domain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: portfolio-service
                port:
                  number: 80
```
Apply the manifests to your cluster:
```
kubectl apply -f portfolio-deployment.yaml
kubectl apply -f portfolio-service.yaml
kubectl apply -f portfolio-ingress.yaml
```
6. Cloudflare Tunnel Setup
Cloudflare Tunnel creates a secure connection from your cluster to Cloudflare's network, allowing you to expose your applications to the internet without direct inbound connections.

Configure Cloudflare Tunnel to point to the NGINX Ingress Controller's external IP (obtained in step 4). Refer to the Cloudflare Tunnel documentation for detailed setup instructions.

Key steps typically include:

Install cloudflared on a node within your cluster (e.g., one of your control plane nodes).
Authenticate cloudflared with your Cloudflare account.
Create a Tunnel.
Configure the Tunnel to route traffic from your desired domain (e.g., www.sr-one.net) to the NGINX Ingress Controller's external IP on port 80.
Update DNS records in Cloudflare to point to your Tunnel.

# Post-Installation Steps
Distribute Kubeconfig to Other Control Plane Nodes
To manage the cluster from your secondary control plane nodes, copy the admin.conf file from the primary control plane:


# Secondary Control Nodes
```
mkdir -p $HOME/.kube
scp mandalore:/etc/kubernetes/admin.conf $HOME/.kube/config # Replace 'mandalore' with the primary control plane's hostname or IP
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Allow Pods on Control Plane Nodes (Optional)
By default, control plane nodes have a taint that prevents pods from being scheduled on them. If you wish to run application pods on your control plane nodes (e.g., for smaller homelabs), you can remove this taint. Use with caution in production environments.
```
kubectl taint nodes goldnine node-role.kubernetes.io/control-plane- || true
kubectl taint nodes chewbacca node-role.kubernetes.io/control-plane- || true
```
# Contributing
Feel free to open issues or pull requests if you have suggestions for improvements or encounter any problems.

# Author
Justyn Larry – @jlarry77
email:  justynlarry@gmail.com

# License
This project is open-sourced under the MIT License (You should create a LICENSE file in your repository with the MIT license text).

# Questions or Feedback?
If you have any questions about this setup or want to provide feedback, please feel free to reach out or open an issue!
