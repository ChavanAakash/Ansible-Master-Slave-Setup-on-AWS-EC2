# Ansible Master–Slave Setup on AWS EC2

A simple, automated practical to configure Ansible on an AWS Ubuntu EC2 Master instance and manage a Slave instance using SSH key authentication.

## Architecture

```text
             AWS EC2
        ┌─────────────────┐
        │   Master Node   │
        │  Ubuntu Server  │
        │     Ansible     │
        └────────┬────────┘
                 │
                 │ SSH (Port 22)
                 │
        ┌────────▼────────┐
        │   Slave Node    │
        │  Ubuntu Server  │
        │  Managed Host   │
        └─────────────────┘
```

## Prerequisites

- An AWS account.
- Two Ubuntu EC2 instances:
  - `ansible-master`
  - `ansible-slave`
- SSH access to both instances.
- Both instances must be able to communicate over SSH.
- An AWS Security Group allowing SSH (port 22) from your trusted IP and Master-to-Slave traffic.
- An EC2 key pair for initial access.

> **Security:** Do not upload private keys, AWS credentials, or other secrets to GitHub.

## Step 1: Launch EC2 Instances

Launch two Ubuntu Server instances from the AWS EC2 console.

| Configuration | Master | Slave |
|---|---|---|
| Instance name | ansible-master | ansible-slave |
| Operating system | Ubuntu Server | Ubuntu Server |
| Role | Runs Ansible | Managed node |
| Connection | SSH | SSH |

Copy the Slave's **Private IPv4 address**. It will be required during setup.

## Step 2: Create the Master Setup Script

Connect to the Master using EC2 Instance Connect as the `ubuntu` user.

Create the script:

```bash
nano master_setup.sh
```

Paste the following script:

```bash
#!/bin/bash
set -e

echo "=== Ansible Master Setup ==="

if [ "$(id -un)" != "ubuntu" ]; then
    echo "Please run this script as the ubuntu user."
    exit 1
fi

sudo apt-get update
sudo apt-get install -y ansible openssh-client

mkdir -p "$HOME/.ssh"
chmod 700 "$HOME/.ssh"

KEY="$HOME/.ssh/ansible_ed25519"

if [ ! -f "$KEY" ]; then
    ssh-keygen -t ed25519 -N "" -f "$KEY"
fi

read -rp "Enter Slave PRIVATE IP: " SLAVE_IP

if [[ ! "$SLAVE_IP" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]]; then
    echo "Invalid IP address."
    exit 1
fi

sudo mkdir -p /etc/ansible

printf '[slaves]\nslave1 ansible_host=%s ansible_user=ubuntu ansible_ssh_private_key_file=%s\n' \
    "$SLAVE_IP" "$KEY" | sudo tee /etc/ansible/hosts >/dev/null

chmod 600 "$KEY"

echo
echo "=== MASTER PUBLIC KEY ==="
cat "$KEY.pub"

echo
echo "NEXT:"
echo "1. Copy the public key above."
echo "2. Add it to the Slave ubuntu user's authorized_keys."

read -rp "Press ENTER after adding the key to Slave..."

echo
echo "=== SSH TEST ==="

ssh -i "$KEY" \
    -o IdentitiesOnly=yes \
    -o ConnectTimeout=10 \
    ubuntu@"$SLAVE_IP" true

echo
echo "=== ANSIBLE PING TEST ==="

ansible slaves -m ping

echo
echo "SUCCESS: Master-Slave configuration completed!"
```

Save the file using `Ctrl + O`, press Enter, then `Ctrl + X`.

## Step 3: Execute the Script

Run the following commands on the Master:

```bash
chmod +x master_setup.sh
./master_setup.sh
```

When prompted, enter the Slave's private IP address.

The script will:

1. Update the Ubuntu package list.
2. Install Ansible and the SSH client.
3. Generate an SSH key pair.
4. Configure the Ansible inventory.
5. Display the Master public key.
6. Wait for you to authorize the key on the Slave.
7. Test SSH connectivity.
8. Run the Ansible ping module.

## Step 4: Configure SSH on the Slave

While the Master script is waiting, open a separate terminal connection to the Slave using EC2 Instance Connect.

Run:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the complete public key displayed by the Master script into a **new line** in `authorized_keys`. Preserve any existing authorized keys.

Save the file, then run:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Return to the Master terminal and press Enter to continue.

> Verify the Slave's SSH host fingerprint before accepting it. If the connection fails, check the Security Group, private IP, username, and key configuration.

## Step 5: Verify the Configuration

The script automatically executes:

```bash
ansible slaves -m ping
```

Expected output:

```text
slave1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

A `pong` response confirms that Ansible can connect to the Slave and execute its ping module successfully.

## Step 6: Run Commands on the Slave

Once the configuration is complete, execute these commands from the Master.

### Check connectivity

```bash
ansible slaves -m ping
```

### Check uptime

```bash
ansible slaves -a "uptime"
```

### Check hostname

```bash
ansible slaves -a "hostname"
```

### Copy a file or directory

For example, to copy `/etc/ansible` from the Master to `/var/ansible` on the Slave:

```bash
ansible slaves -m copy -a "src=/etc/ansible dest=/var" -b
```

The `-b` option enables privilege escalation on the managed host.

## Troubleshooting

| Error | Possible solution |
|---|---|
| `Permission denied (publickey)` | Check `authorized_keys`, SSH username, and private-key permissions. |
| `Connection timed out` | Check Security Group rules, network ACLs, routes, and private IP. |
| `Host key verification failed` | Verify the host fingerprint and correct any stale SSH host-key entry. |
| `Permission denied: /var/...` | Use `-b` for tasks requiring root privileges. |
| Python interpreter warning | Check Python availability on the Slave. This warning does not necessarily indicate failure. |

## Security Notes

- Use the `ubuntu` account rather than enabling root SSH login.
- Restrict inbound SSH access to trusted IP addresses and required security groups.
- Keep the private key private and never commit it to Git.
- Use the Slave's private IP for internal communication when the instances can reach each other privately.
- Review permissions before executing scripts with `sudo`.

## Cleanup

When the practical is complete, stop or terminate the EC2 instances if they are no longer needed. Review attached storage and other resources for possible ongoing charges.

## Learning Outcomes

After completing this practical, you should understand:

- Ansible Master and managed-node architecture.
- SSH public-key authentication.
- Ansible inventory configuration.
- Ansible ad-hoc commands.
- Connectivity verification using the Ansible ping module.
- Remote command execution and file copying.

---

**Result:** Ansible is installed on the Master, the Slave is configured for SSH access, and remote management is verified using Ansible.

## Understanding Ansible Inventory: Group Name vs IP Address

### 1. Using the Inventory Group Name

```bash
ansible slaves -m ping
```

**Why does it work?**

The `/etc/ansible/hosts` inventory file contains a group named `slaves` and a managed host named `slave1`.

Example inventory:

```ini
[slaves]
slave1 ansible_host=172.31.36.56 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/ansible_ed25519
```

- `slaves` is the inventory group.
- `slave1` is the inventory hostname.
- `ansible_host` specifies the Slave's actual IP address.
- `ansible_user` specifies the SSH username.
- `ansible_ssh_private_key_file` specifies the SSH private key path.

Ansible reads this inventory and connects to the managed node using its configured IP, username, and key.

### 2. Using the Inventory Hostname

```bash
ansible slave1 -m ping
```

This command also works because `slave1` is the hostname defined in the inventory.

### 3. Why Can a Direct IP Command Fail?

Incorrect command:

```bash
ansible -m ping 172.31.36.56
```

The target must come before the module options in Ansible's command syntax. Also, the IP address must match a host or pattern in the selected inventory.

Correct general syntax:

```bash
ansible <target> -m ping
```

### 4. Ping a Direct IP Address

To target an IP without relying on the default inventory, use a one-host inline inventory:

```bash
ansible all -i "172.31.36.56," -m ping -u ubuntu --private-key ~/.ssh/ansible_ed25519
```

**Important:** The comma after the IP address is required for a single-host inline inventory.

- `all` selects all hosts in this inline inventory.
- `-i` specifies the inventory.
- `-u ubuntu` specifies the SSH user.
- `--private-key` specifies the SSH private key.

### Summary

| Command | Purpose |
|---|---|
| `ansible slaves -m ping` | Ping all hosts in the `slaves` group |
| `ansible slave1 -m ping` | Ping the host named `slave1` |
| `ansible all -i "172.31.36.56," -m ping -u ubuntu --private-key ~/.ssh/ansible_ed25519` | Ping a direct IP using inline inventory |

**Note:** If `ansible slaves -m ping` returns `pong`, the Master can communicate with the Slave successfully through Ansible.

