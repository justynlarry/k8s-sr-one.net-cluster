### Day 1 - 05/28/2025
- Research and planning for the cluster
- 5 Kubernetes Nodes, 3 Control: 2 of which hosting Web-Pods, and 2 Worker Nodes
- High Availability -> Install Cloudflare and configure tunnel access on each Control
- Thorough revising and restructuring of Order of Operations

### Day 2 - 05/29/2025
- Revisit plan
- Alter original plan from using existing VMs on Proxmox -> Create all new nodes
- Spin up and begin configuring 5 K8s nodes
- Configure automation script: kube_clust_install.sh to install:
	- Docker
	- kubectl, kubeadm, kubelet
	- containerd
  on each of the nodes, but sending it via scp to each and running

### Day 3 - 05/30/2025
- Initialize plan
  - Set up Kube-VIP for shared ip address: 192.168.0.242 on all Control Nodes
  - Set up cluster on Red Two
  - Join Cluster as Control on Gold-Two and Blue-Two
  - Join Cluster as Worker on Gray-Three and Red-Three
  - Install Calico -> CNI Plugin on Red-Two to interact with cluster
  - Had a little trouble joining and getting the cluster up and running, since it was my first time using Kube-VIP, and during set up I made a mistake with the IP address, but was able to resolve it
    and get the cluster up and running.
  - Install MetalLB for load balancing as 'metallb-config.yaml', and configure it to serve the Kubernetes API Server VIP as kube-apiserver-lb.yaml
  - Install NGINX as the Ingress Controller, and verify external IP assigned by MetalLB
  - Docker hub login for access to repository
  - Created Dockerfile to install nginx:alpine, and specify HTML directory -> built and pushed to repository
      - Will have to re-build and push once I have created files for the HTML directory


### Day 4 - 06/02/2025
- Continue executing plan
  - Created manifests for: portfolio-deployment.yaml, portfolio-service.yaml, portfolio-ingress.yaml
  - Applied the manifests
  - Moved red-two's config file to both gold-two and blue-two so that I could taint both control plane nodes, so that they can run the web pods
  - Created a VERY basic web-page and style sheet to show that the application is working on a local level, before configuring cloudflare
