# kubernetes-baremetal-cloudflare-tunnel
Production-grade bare-metal Kubernetes cluster built on laptops using kubeadm and Calico CNI, with applications securely exposed to the internet via Cloudflare Zero Trust Tunnel (no public IP, no open ports).


* ✅ Kubernetes bare-metal
* ✅ Tailscale VPN for node networking
* ✅ Calico CNI
* ✅ NodePort services
* ✅ Cloudflare Tunnel (Zero Trust)
* ✅ Restart-safe & fail-proof
* ✅ Step-by-step installation
* ✅ Recruiter / interview ready


```
# Kubernetes Bare-Metal Cluster with Tailscale, Calico & Cloudflare Zero Trust

> **Production-grade bare-metal Kubernetes cluster built on laptops using kubeadm, with node-to-node networking over Tailscale VPN, Calico CNI for pod networking, and secure internet exposure via Cloudflare Tunnel (Zero Trust) — no public IPs and no open inbound ports.**

---

## 📌 Project Overview

This project demonstrates a **real-world, on-prem / homelab Kubernetes architecture** built without any cloud provider.

Two laptops form a multi-node Kubernetes cluster:
- Securely connected using **Tailscale VPN**
- Using **Calico** for pod networking
- Exposing applications securely to the internet via **Cloudflare Tunnel (Zero Trust)**

This design mirrors how Kubernetes is deployed in **enterprise on-prem, edge, and hybrid environments**.

---

## 🧠 Architecture



Internet
│
Cloudflare DNS + Zero Trust
│
Cloudflare Tunnel
│
cloudflared (Control Plane Node)
│
NodePort Service
│
Kubernetes Service
│
Pods (Load Balanced Across Nodes)
│
Calico CNI (Pod Networking)
│
Tailscale VPN (Encrypted Node-to-Node Connectivity)
│
Bare-Metal Nodes (Laptops)



---

## 🖥️ Infrastructure

| Role | Machine | OS |
|----|----|----|
| Control Plane + Worker | Laptop 1 | Ubuntu Linux |
| Worker Node | Laptop 2 | Ubuntu Linux |

---

## ⚙️ Technology Stack

- **Kubernetes:** kubeadm
- **Container Runtime:** containerd
- **CNI:** Calico
- **Node Networking:** Tailscale VPN (WireGuard)
- **Service Exposure:** NodePort
- **Public Access:** Cloudflare Tunnel (Zero Trust)
- **OS:** Ubuntu Linux

---

## 🔐 Networking Design

| Layer | Technology | Purpose |
|----|----|----|
| Node-to-Node | **Tailscale VPN** | Encrypted cluster connectivity |
| Pod Networking | **Calico CNI** | Pod-to-pod & cross-node routing |
| Service Exposure | **NodePort** | Internal cluster access |
| Internet Access | **Cloudflare Tunnel** | Secure public exposure |

All Kubernetes communication happens over **Tailscale IPs (100.x.x.x)**.

---

# 🔧 Step-by-Step Installation Guide

---

## 🟢 Prerequisites

- 2 Linux machines (laptops / bare-metal)
- Ubuntu 20.04+
- Internet access
- Sudo privileges
- Cloudflare account (for tunnel)

---

## 🟢 Step 1: Install & Configure Tailscale (ALL NODES)

### Install Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
````

### Authenticate

```bash
sudo tailscale up
```

Verify:

```bash
tailscale status
```

Each node will receive a **100.x.x.x** IP.
👉 These IPs are used for Kubernetes communication.

---

## 🟢 Step 2: Base System Preparation (ALL NODES)

### Disable Swap (Mandatory)

```bash
sudo swapoff -a
sudo sed -i '/swap.img/d' /etc/fstab
```

### Load Kernel Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Persist:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

### Sysctl Configuration

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                = 1
EOF

sudo sysctl --system
```

---

## 🟢 Step 3: Install Container Runtime (containerd) – ALL NODES

```bash
sudo apt update
sudo apt install -y containerd
```

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 🟢 Step 4: Install Kubernetes Components (ALL NODES)

```bash
sudo apt install -y apt-transport-https ca-certificates curl
```

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo tee /etc/apt/keyrings/kubernetes.asc
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes.asc] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

---

## 🟢 Step 5: Initialize Control Plane (MASTER NODE)

Use the **Tailscale IP** of the master:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 🟢 Step 6: Install Calico CNI

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Verify:

```bash
kubectl get pods -n kube-system
```

---

## 🟢 Step 7: Join Worker Node

On the worker node, run the join command (using **Tailscale IP**):

```bash
sudo kubeadm join <MASTER_TAILSCALE_IP>:6443 --token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```

Verify:

```bash
kubectl get nodes
```

---

## 🟢 Step 8: Deploy Application

```bash
kubectl create namespace himanshu-portfolio
```

```bash
kubectl apply -f manifests/deployment.yaml
kubectl apply -f manifests/service.yaml
```

Service Type:

```
NodePort (32527)
```

---

## 🟢 Step 9: Access Application Internally

```text
http://<NODE_TAILSCALE_IP>:32527
```

NodePort works from **any node**.

---

## 🟢 Step 10: Expose Application via Cloudflare Tunnel

Install cloudflared:

```bash
sudo apt install cloudflared
```

Run tunnel:

```bash
cloudflared tunnel run --token <TUNNEL_TOKEN>
```

Tunnel forwards traffic to:

```text
http://<NODE_TAILSCALE_IP>:32527
```

Mapped to:

```
https://your-domain
```

---

## 🔁 Restart & Fail-Proof Validation

Reboot nodes:

```bash
sudo reboot
```

After restart:

```bash
kubectl get nodes
kubectl get pods -A
```

✔ kubelet auto-starts
✔ containerd auto-starts
✔ Pods recover
✔ Tunnel reconnects

---

## 🔐 Security Highlights

* Zero inbound ports exposed
* Encrypted WireGuard networking (Tailscale)
* Zero Trust internet access (Cloudflare)
* No public IPs anywhere

---

## 📈 What This Project Demonstrates

* Bare-metal Kubernetes (no cloud dependency)
* Secure overlay networking with Tailscale
* Production-grade pod networking (Calico)
* Zero Trust service exposure
* Real-world Kubernetes troubleshooting
* On-prem / edge Kubernetes architecture

---

## 🧑‍💻 Author

**Himanshu Saini**
DevOps | Kubernetes | Cloud | Linux

🌐 Portfolio: [https://portfolio.himanshusaini.online](https://portfolio.himanshusaini.online)
🐙 GitHub: [https://github.com/HimanshuSaini70](https://github.com/HimanshuSaini70)

---

## 🚀 Future Enhancements

* Calico NetworkPolicies
* HPA (Horizontal Pod Autoscaler)
* Helm charts
* GitOps (ArgoCD)
* Prometheus & Grafana monitoring

---

⭐ If you found this project useful, give it a star!

```

```
