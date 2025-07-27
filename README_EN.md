# Detailed Guide for Running Kubernetes_ansible_setup

## What is this?

This project is an automated Ansible playbook that will deploy a Kubernetes (RKE2) cluster on multiple servers for you.  
You don't need to install Kubernetes manually — Ansible will do everything step by step as described in the playbook.

---

## What do you need?

- **A computer/server to run Ansible from** (any Linux, e.g. Ubuntu, is fine)
- **SSH access** to your future Kubernetes servers (root or a user with sudo)
- **Python 3.6+** on your machine
- **pip** (Python package manager)
- **git** (to clone the repository)

---

## Step 1. Clone the project

```bash
git clone https://github.com/Wonka1124/Kubernetes_ansible_setup
cd Kubernetes_ansible_setup-main
```

**Explanation:**
- This command will download the project from GitHub and go to the right folder.
- If you don't have git, install it: `sudo apt install git`

**Example output:**
```
Cloning into 'Kubernetes_ansible_setup'...
remote: Enumerating objects: ...
Receiving objects: ...
Resolving deltas: ...
```

---

## Step 2. Install Ansible and dependencies

1. Install Ansible:
   ```bash
   sudo apt install ansible
   ```

   **Example output:**
   ```
   Successfully installed ansible ...
   ```

2. Install required Ansible collections:
   ```bash
   ansible-galaxy collection install -r collections/requirements.yaml
   ```
   **Explanation:**
   - This command downloads all modules needed to work with Kubernetes and Linux.

   **Example output:**
   ```
   Starting collection install process
   Process install dependency map
   Installing 'ansible.utils:...'
   Installing 'community.general:...'
   Installing 'ansible.posix:...'
   Installing 'kubernetes.core:...'
   ```

---

## Step 3. Prepare your server list (inventory)

Open the file `inventory/hosts.ini` in any text editor.

Example:
```
[servers]
server1 ansible_host=192.168.3.21
server2 ansible_host=192.168.3.22
server3 ansible_host=192.168.3.23

[agents]
agent1 ansible_host=192.168.3.24
agent2 ansible_host=192.168.3.25

[rke2:children]
servers
agents

[rke2:vars]
ansible_user=ansible
```

**Explanation:**
- **servers** — control-plane (master) nodes of Kubernetes.
- **agents** — worker nodes.
- **ansible_user** — the user Ansible will use to connect via SSH (must exist on all servers and have sudo rights).

**Check SSH access:**
```bash
ssh ansible@192.168.3.21
```
**Example output:**
```
Welcome to Ubuntu 22.04 LTS ...
ansible@server1:~$
```

---

## Step 4. Configure variables

Open the file `inventory/group_vars/all.yaml` and change the parameters for your network:

```yaml
vip: 192.168.3.50                # Virtual IP for cluster access
metallb_version: v0.13.12        # Version of metallb (load balancer)
lb_range: 192.168.3.80-192.168.3.90  # IP range for the load balancer
lb_pool_name: first-pool          # Pool name for metallb
```

**Explanation:**
- **vip** — the IP address your cluster will be available at (must be free in your network).
- **lb_range** — IP range for the load balancer (must be free).

---

## Step 5. Check server availability via Ansible

```bash
ansible -i inventory/hosts.ini all -m ping
```
**Explanation:**
- This command checks that Ansible can connect to all servers from the list.

**Example output:**
```
server1 | SUCCESS => {"changed": false, "ping": "pong"}
server2 | SUCCESS => {"changed": false, "ping": "pong"}
server3 | SUCCESS => {"changed": false, "ping": "pong"}
agent1 | SUCCESS => {"changed": false, "ping": "pong"}
agent2 | SUCCESS => {"changed": false, "ping": "pong"}
```

If you see SUCCESS — everything is ready!

---

## Step 6. Run the playbook

1. Go to the playbook folder:
   ```bash
   cd Kubernetes_ansible_setup-main
   ```

2. Run the main playbook:
   ```bash
   ansible-playbook -i inventory/hosts.ini site.yaml
   ```

**Explanation:**
- Ansible will connect to each server via SSH and perform all steps to install and configure Kubernetes.

**Example output:**
```
PLAY [Prepare all nodes] ****************************************************************
TASK [prepare-nodes : ...] *************************************************************
changed: [server1]
changed: [server2]
changed: [server3]
...
PLAY [Deploy Kube VIP] *****************************************************************
TASK [kube-vip : ...] ******************************************************************
ok: [server1]
ok: [server2]
...
PLAY RECAP ****************************************************************************
server1 : ok=20 changed=15 unreachable=0 failed=0
server2 : ok=20 changed=15 unreachable=0 failed=0
...
```

- **ok=20** — how many tasks were successful
- **changed=15** — how many tasks made changes
- **failed=0** — no errors

---

## What will happen?

- Ansible will connect to each server via SSH.
- Prepare the servers (update, install required packages).
- Download and install RKE2 (Kubernetes).
- Set up the virtual IP (kube-vip).
- Add all servers and agents to the cluster.
- Install the metallb load balancer.

---

## How to check if the cluster is working?

1. On a control-plane server (e.g., server1), find the kubeconfig file:
   ```bash
   sudo cat /etc/rancher/rke2/rke2.yaml
   ```
2. Copy this file to your computer and set the environment variable:
   ```bash
   export KUBECONFIG=~/rke2.yaml
   kubectl get nodes
   ```
**Example output:**
```
NAME      STATUS   ROLES                  AGE   VERSION
server1   Ready    control-plane,master   10m   v1.29.4
server2   Ready    control-plane,master   10m   v1.29.4
server3   Ready    control-plane,master   10m   v1.29.4
agent1    Ready    <none>                 8m    v1.29.4
agent2    Ready    <none>                 8m    v1.29.4
```

---

## Useful tips

- **User rights:** The user Ansible connects as must have sudo rights (or be root).
- **SSH:** If a password is required, you can add the `-k` flag (for user password) and `--ask-become-pass` (for sudo password).
- **Errors:** If something goes wrong — carefully read the error messages in the console. Usually, they explain what is wrong (no access, not enough rights, package not found, etc.).
- **Security:** Do not use root/simple passwords in production!

---

## FAQ

**Q: Do I need to install anything on the servers in advance?**  
A: No, just make sure SSH access and a user with sudo are available.

**Q: Can I run this from Windows?**  
A: It is recommended to use Linux or WSL (Windows Subsystem for Linux).

**Q: How do I check if the cluster is working?**  
A: After the playbook finishes, the kubeconfig file will appear on the control-plane servers (usually `/etc/rancher/rke2/rke2.yaml`). You can copy it and use it with kubectl.

---

## Useful commands

- Check server availability:
  ```bash
  ansible -i inventory/hosts.ini all -m ping
  ```
- Rerun the playbook after fixing errors:
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yaml
  ```

---

## If something doesn't work

- Check SSH access.
- Check user rights.
- Check that IP addresses and variables are correct.
- Carefully read error messages.
