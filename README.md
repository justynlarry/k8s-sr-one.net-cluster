# k8s-sr-one.net-cluster
Five Node Kubernetes Cluster, using Docker, MetalLB, Kubernetes, Cloudflare, and NGINX.

# Kubernetes Homelab Cluster

Documentation for the step-by-step setup of a highly-available Kubernetes cluster across Proxmox VMs, using MetalLB for load balancing and NGINX as an ingress controller.

## Topology

| Node       | VM         | Role        |
|------------|------------|-------------|
| red-one    | mandalore  | Control     |
| red-one    | falcon     | Worker      |
| secondsun  | goldnine   | Control     |
| hth        | kashyyk    | Worker      |
| deathStar  | shire      | Control     |

## Components

- MetalLB: Bare-metal load balancer
- NGINX Ingress Controller
- Cloudflare Tunnel
- Custom Dockerized Web App

## File Structure

...

## Goals

✅ High-availability Control Plane  
✅ Bare-metal Load Balancer  
✅ External Ingress via Cloudflare  
✅ GitOps-style deployment manifests

## Author

Justyn Larry – [@jlarry77](https://github.com/jlarry77)
