# 🛠️ Node Configuration
This guide details the configuration steps for setting up a Raspberry Pi 4 (8GB) as a homelab node

## 📋 Prerequisites
- **Hardware:** Raspberry Pi 4 (8GB) running Raspberry Pi OS Lite (64-bit).
- **Network:** Static IP address (192.168.68.10 in this guide).
- **Storage:** External drive (e.g., USB SSD or HDD) for persistent storage.
- **Local Machine:** A system with SSH and Ansible installed for remote management.

## 🔑 1. Configure SSH Access
Set up passwordless SSH access to manage the Raspberry Pi remotely.

### Steps
1. **Generate an SSH key:** (if not already done)
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```
Follow prompts to create a keypair (`~/.ssh/id_ed25519` and `~/.ssh/id_ed25519.pub`).

2. **Copy the public key** to the Raspberry Pi:
```bash
ssh-copy-id tristan@192.168.68.10
```
Enter the user's password when prompted. This adds the public key to `~/.ssh/authorized_keys` on the Pi.

3. **Verify SSH access:**
```bash
ssh tristan@192.168.68.10
```
You should log in without entering a password.

### Notes
Optionally add the following to your SSH config (`~/.ssh/config`) on your local machine for convenience
```bash
Host pi
    HostName 192.168.68.10
    User tristan
```
Now the Pi is accesible via `ssh pi`

## 🛠️ 2. Install and Verify Ansible
Ansible automates the K3s installation. Install it on your local machine (not the Pi).

### Steps
1. **Install Ansible:** (Arch Linux)
```bash
sudo pacman -S ansible
```

2. **Verify the installation:**
```bash
ansible -v
```

### Notes
- Ensure Python is installed on both the local machine and the Pi, as Ansible requires it.
- Ansible uses SSH, so Step 1 must be complete

## 🚀 3. Install K3s with Ansible Playbook'
Use an Ansible playbook to deploy K3s on the Raspberry Pi as a single-node Kubernetes cluster.

### Steps
1. **Navigate to the Ansible configuration directory:**
```bash
cd configuration/ansible
```

2. **Run the K3s playbook:**
```bash
ansible-playbook k3s.yml --inventory inventory.ini
```

### Playbook Details
The `k3s.yml` playbook references the `k3s-install` role, which performs the following tasks:
- **Disables swap:** Modifies `/etc/dphys-swapfile` to set `CONF_SWAPSIZE=0` for Kubernetes compatibility.
- **Enables cgroups:** Adds `cgroup_memory=1 cgroup_enable=memory` to `/boot/firmware/cmdline.txt` if not present, followed by a reboot if modified.
- **Installs K3s:** Downloads and runs the K3s installation script (`curl -sfL https://get.k3s.io | sh -`) with `--write-kubeconfig-mode 644` for accessible permissions.
- **Enables K3s service:** Ensures the `k3s` systemd service is started and enabled.
- **Fetches kubeconfig:** Copies `/etc/rancher/k3s/k3s.yaml` from the Pi to `~/.kube/k3s-config.yaml` on the local machine.

### Notes
- The playbook assumes a single-node K3s cluster, combining control plane and worker roles.
- If the playbook fails, check SSH connectivity and Python availability on the Pi

## ⚙️ 4. Update kubeconfig File
Configure your local machine to interact with the K3s cluster.

1. **Copy the kubeconfig file** (handled by the playbook)

2. **Update the server address:** Edit `~/.kube/k3s-config.yaml` to set the `server` field to the Pi’s IP:
```bash
clusters:
- cluster:
    server: https://192.168.68.10:6443
    ...
```
Use `vim`, `nano`, or another editor.

3. **Set the KUBECONFIG environment variable:**
```bash
export KUBECONFIG=~/.kube/k3s-config.yaml
```
Add this to `~/.bashrc` or `~/.zshrc` for persistence.

4. **Verify cluster access:**
```bash
kubectl get nodes
```
You should see the Pi listed as a node.

### Notes
If using multiple clusters, merge `k3s-config.yaml` into `~/.kube/config` using a tool like `kubectl config`.

## 💾 5. Set Up External Drive

### Steps
1. **Create a new partition:**
```bash
sudo fdsik /dev/sda
```
- Press `n` to create a new partition, select defaults for a single partition.
- Press `w` to write changes and exit.

2. **Format the partition** as ext4:
```bash
sudo mkfs.ext4 /dev/sda1
```

3. **Mount the partition:**
- Find the parition's UUID:
```bash
sudo mkdir -p /mnt/data
```

- Edit `/etc/fstab`
```bash
sudo vim /etc/fstab
```

- Add the following line:
```
UUID=16c6cdf3-9bcb-4be4-91a9-97014d4f0bce /mnt/data ext4 defaults,nofail 0 2
```

- Verify the mount:
```bash
sudo mount -a
df -h /mnt/data
```

### Notes
- Replace `/dev/sda` with the actual device name (check with `lsblk`).
- The `nofail` option ensures the Pi boots even if the drive is disconnected.
- Use the external drive for Kubernetes persistent volumes by configuring storage classes to use `/mnt/data`.

## 🔍 Troubleshooting
- **SSH issues:** Verify the Pi’s IP, user credentials, and SSH service status (`sudo systemctl status ssh`).
- **Ansible failures:** Check SSH connectivity, Python installation, and playbook syntax.
- **K3s errors:** Inspect logs with `sudo journalctl -u k3s`.
- **kubectl errors:** Ensure `KUBECONFIG` is set and the server IP is correct.