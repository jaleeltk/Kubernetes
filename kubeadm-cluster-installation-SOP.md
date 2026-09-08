# Kubeadm Kubernetes Cluster Installation — Standard Operating Procedure

**Purpose:** Build a working 2-node Kubernetes cluster (1 control-plane + 1 worker) using kubeadm on Ubuntu 22.04, for CKA exam practice.

**Environment used in this SOP:**
- Control plane node: `master1` — `192.168.1.51`
- Worker node: `worker1` — `192.168.1.52`
- OS: Ubuntu 22.04 Server
- Kubernetes version: v1.32
- Container runtime: containerd
- Pod network: Calico

Adjust IP addresses/hostnames to match your own environment if different.

---

## Part 0 — Before You Start

Both VMs need at minimum:
- **2 CPU cores** (kubeadm's control-plane preflight check fails below this — a 1-core VM will not pass)
- **2 GB RAM minimum** (4 GB recommended, especially for the control-plane node)
- A **static IP address** set during OS install (not DHCP) — an IP that changes later is what breaks a cluster after a network change

---

## Part 1 — Fix Hostname Resolution (Both Nodes)

**What this does:** Kubernetes nodes need to reliably resolve each other's hostnames to real IP addresses. Ubuntu, by default, maps your own hostname to `127.0.1.1` (loopback), which silently breaks this — the fix below overrides that.

1. Open the hosts file:
   ```
   sudo nano /etc/hosts
   ```
2. Find the line `127.0.1.1 <hostname>` and put a `#` in front of it to disable it.
3. Add two lines mapping both node hostnames to their real static IPs:
   ```
   192.168.1.51 master1
   192.168.1.52 worker1
   ```
4. Save (Ctrl+O, Enter) and exit (Ctrl+X).
5. Repeat this exact edit on **both** nodes — both files should contain both hostname entries.
6. Verify it worked:
   ```
   ping master1
   ```
   It should resolve to the real IP (`192.168.1.51`), not `127.0.1.1`. Press Ctrl+C to stop the ping.

---

## Part 2 — Disable Swap (Both Nodes)

**What this does:** Kubelet refuses to start if swap is active. This must be turned off both right now, and permanently (so it doesn't come back after a reboot).

1. Turn swap off immediately:
   ```
   sudo swapoff -a
   ```
2. Prevent it turning back on after reboot — open the fstab file:
   ```
   sudo nano /etc/fstab
   ```
3. Find the line referencing swap (commonly `/swap.img none swap sw 0 0`) and put a `#` at the very start of that line.
4. Save and exit.
5. **Double-check it actually worked** — this step is easy to silently fail:
   ```
   cat /etc/fstab
   ```
   Confirm the swap line clearly starts with `#`.

---

## Part 3 — Load Kernel Modules and Network Settings (Both Nodes)

**What this does:** Kubernetes networking depends on two kernel modules being loaded (`overlay`, `br_netfilter`), and specific network settings being enabled so traffic can pass correctly between pods.

1. Load the modules immediately:
   ```
   sudo modprobe overlay
   sudo modprobe br_netfilter
   ```
2. Make them load automatically on every future boot:
   ```
   sudo nano /etc/modules-load.d/k8s.conf
   ```
   Type these two lines, then save and exit:
   ```
   overlay
   br_netfilter
   ```
3. Set the required network settings:
   ```
   sudo nano /etc/sysctl.d/k8s.conf
   ```
   Type these three lines, then save and exit:
   ```
   net.bridge.bridge-nf-call-iptables = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward = 1
   ```
4. Apply the settings immediately:
   ```
   sudo sysctl --system
   ```

---

## Part 4 — Install containerd (Both Nodes)

**What this does:** containerd is the container runtime — the actual engine that pulls images and runs containers. Kubernetes needs one; we're using containerd.

1. Update package lists:
   ```
   sudo apt update
   ```
2. Install containerd:
   ```
   sudo apt install -y containerd
   ```
3. Create its config folder:
   ```
   sudo mkdir -p /etc/containerd
   ```
4. Generate its default configuration file:
   ```
   containerd config default | sudo tee /etc/containerd/config.toml
   ```
5. Fix the cgroup driver setting (critical — without this, kubeadm init fails later with a cgroup mismatch error):
   ```
   sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
   ```
6. Restart and enable containerd so it starts on every boot:
   ```
   sudo systemctl restart containerd
   sudo systemctl enable containerd
   ```

---

## Part 5 — Install kubeadm, kubelet, kubectl (Both Nodes)

**What this does:** Installs the three core Kubernetes tools — `kubeadm` (builds the cluster), `kubelet` (the agent that runs on every node), `kubectl` (the command-line tool you use to control the cluster).

1. Install prerequisite packages:
   ```
   sudo apt install -y apt-transport-https ca-certificates curl gpg
   ```
2. Create a folder for the Kubernetes package signing key:
   ```
   sudo mkdir -p /etc/apt/keyrings
   ```
3. Download and save the signing key:
   ```
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```
4. Add the Kubernetes package repository:
   ```
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
5. Update package lists again so the new repository is recognized:
   ```
   sudo apt update
   ```
6. Install the three tools:
   ```
   sudo apt install -y kubelet kubeadm kubectl
   ```
7. Lock their versions so a routine system update doesn't accidentally upgrade them out of sync with each other:
   ```
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

> **Known pitfall:** If you run the `apt-mark hold` command before the packages are actually installed (e.g., due to an earlier failed install), a later `apt install` attempt will fail with *"Held packages were changed and -y was used without --allow-change-held-packages."* If this happens, re-run the install with:
> ```
> sudo apt install -y --allow-change-held-packages kubelet kubeadm kubectl
> ```

---

## Part 6 — Initialize the Cluster (Control-Plane Node ONLY)

**What this does:** This is the main step — it actually creates the Kubernetes control plane (API server, etcd, scheduler, controller-manager) on this one node.

**Run this ONLY on `master1`, never on the worker.**

1. Initialize the cluster:
   ```
   sudo kubeadm init --apiserver-advertise-address=192.168.1.51 --pod-network-cidr=192.168.0.0/16
   ```
   - `--apiserver-advertise-address` tells the cluster which IP other nodes should use to reach the API server.
   - `--pod-network-cidr` reserves an internal IP range for pod-to-pod networking (must match what your chosen CNI plugin expects — `192.168.0.0/16` is Calico's default).
   - This takes a few minutes.
   - **At the very end, it prints a `kubeadm join ...` command with a token and hash. Copy this entire command and save it — you will need it for every worker node.**

2. Set up `kubectl` access for your regular user:
   ```
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

> **Known pitfall:** If `kubeadm init` fails partway through (e.g., due to swap still being on, or insufficient CPU), you must clean up before retrying:
> ```
> sudo kubeadm reset -f
> ```
> Then fix the underlying issue and re-run `kubeadm init`.

---

## Part 7 — Install a Pod Network / CNI Plugin (Control-Plane Node ONLY)

**What this does:** Without this, pods cannot talk to each other and nodes will stay stuck in `NotReady` forever. We're using Calico.

1. Apply the Calico manifest:
   ```
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
   ```
2. Confirm the control-plane node is now `Ready`:
   ```
   kubectl get nodes
   ```

---

## Part 8 — Join the Worker Node

**What this does:** Connects `worker1` to the cluster so it can actually run workloads.

**Run this ONLY on `worker1`**, using the exact join command that was printed at the end of Part 6:
```
sudo kubeadm join 192.168.1.51:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-hash>
```

If you lost the original join command, regenerate it from the control-plane node:
```
kubeadm token create --print-join-command
```

---

## Part 9 — Final Verification

**Run on `master1`:**
```
kubectl get nodes
```

Expected result — both nodes showing `Ready` (the worker may briefly show `NotReady` for 30-60 seconds right after joining while Calico finishes setting up its networking there — this is normal, just re-check after a short wait):

```
NAME      STATUS   ROLES           AGE   VERSION
master1   Ready    control-plane   ...   v1.32.13
worker1   Ready    <none>          ...   v1.32.13
```

---

## Common Issues Encountered (Troubleshooting Reference)

| Symptom | Cause | Fix |
|---|---|---|
| `preflight] [ERROR NumCPU]` | VM has fewer than 2 CPU cores | Increase VM's CPU count in Proxmox, reboot, retry |
| `kubelet` crash-loops, journal shows `"running with swap on is not supported"` | Swap re-enabled after reboot (fstab edit didn't persist) | Re-check `/etc/fstab`, ensure swap line is genuinely commented out, `swapoff -a`, restart kubelet |
| `apt install` fails: `Held packages were changed` | Ran `apt-mark hold` before the packages were actually installed | Re-run install with `--allow-change-held-packages` |
| `ping <hostname>` resolves to `127.0.1.1` instead of the real IP | Ubuntu's auto-generated `127.0.1.1 <hostname>` line in `/etc/hosts` takes priority | Comment out that line, keep only the correct static-IP mapping |
| Node stuck in `NotReady` indefinitely | No CNI/pod network installed yet | Apply the Calico manifest (Part 7) |
| `kubeadm: command not found` after apparently successful install | Install silently didn't complete (often due to the held-package issue above) | Verify with `which kubeadm`; re-run install if empty |

---

*Document generated from a live Ubuntu 22.04 kubeadm cluster build — every step above was actually run and verified working.*
