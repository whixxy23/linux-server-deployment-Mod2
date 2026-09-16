# Linux Server Deployment — Module 2

Deploying and securing a Linux server on AWS EC2: from a bare VM to a running, tested, internet-facing service.

**Flow:** Create VM → Configure Linux → Secure Access → Deploy Service → Test Connectivity

---

## Linux Fundamentals — Cheat Sheet

### File permissions
```bash
ls -l                     # view permissions (rwxr-xr-x owner/group/other)
chmod 755 file.sh         # rwx for owner, rx for group/other
chmod u+x file.sh         # add execute for owner only
chown user:group file.sh  # change owner and group
```
Permission digits: 4=read, 2=write, 1=execute (sum them — 7=rwx, 5=r-x, 4=r--).

### Users & groups
```bash
sudo useradd -m -s /bin/bash deploy   # create user with home dir
sudo passwd deploy                    # set password
sudo usermod -aG sudo deploy          # add to sudo group
groups deploy                         # check group membership
cat /etc/passwd                       # list all users
```

### Process management
```bash
ps aux                    # list all running processes
top                        # live process monitor (q to quit)
kill -9 <PID>              # force-kill a process
systemctl status nginx     # check a service's status
systemctl start/stop/restart nginx
systemctl enable nginx     # start automatically on boot
journalctl -u nginx -f     # follow a service's logs
```

### SSH keys
```bash
ssh-keygen -t ed25519 -C "wisdomntekim23@gmail.com"   # generate a key pair
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@172.31.30.45  # push public key to server
ssh -i ~/.ssh/id_ed25519 user@172.31.30.45              # connect using the key
```

---

## Networking Fundamentals

- **IP addressing / CIDR** — `10.0.0.0/24` means the first 24 bits are the network portion, leaving 256 addresses. AWS VPCs and subnets are defined this way.
- **DNS** — A records map a domain to an IPv4 address; CNAME records alias one domain to another.
- **Ports** — 22 (SSH), 80 (HTTP), 443 (HTTPS) are the ones you'll touch here. A port is only reachable if it's open at *every* layer between client and server.
- **Firewalls vs. Security Groups** — a Security Group is a stateful, instance-level virtual firewall in AWS (allow rules only, return traffic auto-permitted). A host firewall like `ufw` is a second, OS-level layer of the same idea. Both were configured in this project — defense in depth.

---

## Deployment Log

### 1. Create VM
- Provider: AWS EC2
- AMI: Ubuntu 24.04 LTS
- Instance type: t3.micro
- Region: us-east 1

### 2. Configure Linux
```bash
sudo apt update && sudo apt upgrade -y
sudo useradd -m -s /bin/bash deploy
sudo usermod -aG sudo deploy
```

### 3. Secure access
```bash
# On local machine — generate and copy key
ssh-keygen -t ed25519 -C "deploy-key"
ssh-copy-id -i ~/.ssh/id_ed25519.pub deploy@172.31.30.45

# On server — harden SSH
sudo nano /etc/ssh/sshd_config
#   PasswordAuthentication no
#   PermitRootLogin no
sudo systemctl restart sshd
```
- Security group inbound rules: `[SSH (22) restricted to my IP only, HTTP (80) open to 0.0.0.0/0]`

### 4. Deploy service
```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 5. Test connectivity
```bash
curl localhost                    # confirm nginx responds locally
curl http://100.59.207.180    # confirm it's reachable externally
ssh deploy@172.31.30.45            # confirm key-only login works
ssh -o PreferredAuthentications=password deploy@172.31.30.45  # should FAIL
```
## Results
markdown ![Host firewall (ufw) allowing OpenSSH and port 80 only](screenshots/ufw-status.png)
markdown ![Nginx service active and enabled on boot](screenshots/nginx-status.png)
markdown ![Browser test showing deployed nginx page](screenshots/browser-test.png)

---

## Key Learnings
- The distinction between a stateful AWS security group and a host-level Linux firewall — and why both matter.
- SSH key-based auth removes an entire class of brute-force risk compared to password auth.
- `systemctl enable` vs `start` — one persists across reboots, the other doesn't.
