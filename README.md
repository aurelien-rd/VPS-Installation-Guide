# VPS Security Guide
## Secure method without risk of access loss

This guide presents a step-by-step procedure to effectively secure your VPS server while avoiding common errors that could block your access.

## Table of Contents
- [1. System Update](#1-system-update)
- [2. Creating a User with Sudo Privileges](#2-creating-a-user-with-sudo-privileges)
- [3. Installing and Configuring the Firewall (UFW)](#3-installing-and-configuring-the-firewall-ufw)
- [4. Configuring SSH Key Authentication](#4-configuring-ssh-key-authentication)
- [5. Changing the SSH Port](#5-changing-the-ssh-port)
- [6. Disabling Password Authentication](#6-disabling-password-authentication)
- [7. Disabling Root Login via SSH](#7-disabling-root-login-via-ssh)
- [8. Finalizing Firewall Configuration](#8-finalizing-firewall-configuration)
- [9. Installing and Configuring Fail2ban](#9-installing-and-configuring-fail2ban)
- [Best Practices and Precautions](#best-practices-and-precautions)

## 1. System Update

Connect to your VPS as `root` and update the entire system:

```bash
apt update && apt upgrade -y
```

## 2. Creating a User with Sudo Privileges

Create a new administrative user and add them to the sudo group:

```bash
adduser your_username
usermod -aG sudo your_username
```

Verify that sudo privileges work correctly:

```bash
su - your_username
sudo whoami
```

The command should return `root` if the configuration is correct.

## 3. Installing and Configuring the Firewall (UFW)

UFW (Uncomplicated Firewall) is a simple but effective firewall:

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw enable
```

Check the current status of the firewall:

```bash
sudo ufw status
```

## 4. Configuring SSH Key Authentication

### On your local machine:

Generate a new SSH key pair (if you don't already have one):

```bash
ssh-keygen -t ed25519 -C "your_email"
```

Transfer your public key to the server:

```bash
ssh-copy-id your_username@server_ip_address
```

### Verifying Key Authentication

Open a new terminal window and test the connection:

```bash
ssh your_username@server_ip_address
```

You should be able to connect without entering a password.

## 5. Changing the SSH Port

> ⚠️ **IMPORTANT**: Never close your active SSH session before verifying that you can connect with the new settings.

### 5.1. Modify the SSH Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Find and modify the line concerning the port (or add it):

```
Port 2222  # Replace 2222 with the port of your choice
```

### 5.2. Modify the SSH Socket Configuration (for systemd)

```bash
sudo nano /usr/lib/systemd/system/ssh.socket
```

Change `ListenStream=22` to `ListenStream=2222` (or your chosen port)

### 5.3. Apply the Changes

```bash
# Disable and stop the current socket
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket

# Enable and restart the standard SSH service
sudo systemctl enable ssh
sudo systemctl restart ssh

# Reload the systemd configuration
sudo systemctl daemon-reload
```

### 5.4. Allow the New Port in the Firewall

```bash
sudo ufw allow 2222/tcp
```

### 5.5. Verify the Configuration

```bash
sudo ss -tulpn | grep ssh
```

Verify that SSH is listening on your new port.

### 5.6. Test the Connection

Open a new terminal window and test the connection on the new port:

```bash
ssh -p 2222 your_username@server_ip_address
```

## 6. Disabling Password Authentication

Once key authentication is verified, disable passwords:

```bash
sudo nano /etc/ssh/sshd_config
```

Modify or add this line:

```
PasswordAuthentication no
```

Restart the SSH service:

```bash
sudo systemctl restart ssh
```

## 7. Disabling Root Login via SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Modify or add this line:

```
PermitRootLogin no
```

Restart the SSH service:

```bash
sudo systemctl restart ssh
```

## 8. Finalizing Firewall Configuration

Once the new SSH port has been tested and is working, remove the old port:

```bash
sudo ufw delete allow 22/tcp
```

Configure the necessary ports for your services:

```bash
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw allow 8000/tcp # Python/Uvicorn (if necessary)
sudo ufw allow 53       # DNS (tcp and udp)
sudo ufw allow 68/udp   # DHCP client (if necessary)
```

Verify the complete configuration:

```bash
sudo ufw status
```

## 9. Installing and Configuring Fail2ban

Fail2ban monitors system logs and temporarily blocks IP addresses that attempt too many failed connections.

### Installation

```bash
sudo apt update
sudo apt install fail2ban -y
```

### Configuration

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Adjust the following parameters:

```ini
[DEFAULT]
# Ban time (in seconds - here 1 hour)
bantime = 3600
# Observation time (in seconds)
findtime = 600
# Number of attempts before banning
maxretry = 5

[sshd]
enabled = true
port = 2222 # Your current SSH port
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

### Activating the Service

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

### Verification

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

## Best Practices and Precautions

- **Backup Session**: Always keep your SSH session active when testing new settings.
- **Connection Tests**: Test each modification in a new terminal window before proceeding.
- **Emergency Access**: Note that most VPS providers offer emergency access through their administration interface.
- **Key Backup**: Keep your private SSH key in a secure place and consider having a backup copy.
- **Principle of Least Privilege**: Only open ports that are strictly necessary for your services.
- **Monitoring**: Regularly check Fail2ban logs to detect intrusion attempts.

By methodically following these steps in the order indicated, you significantly strengthen the security of your VPS while maintaining reliable access.
