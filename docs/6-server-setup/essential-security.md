---
layout: default
title: Essential Server Security
nav_order: 1
parent: Server Setup
---

# Essential Server Security

When you first get access to a blank Linux server, there are several critical security steps you must take before doing anything else. This guide will walk you through each step to ensure your server is properly secured.

## Prerequisites

- A fresh Linux server (I recommended DigitalOcean provided but it's up to you where you get it)
- Root access to the server
- A local computer with terminal/command prompt access

## Step 1: Initial SSH Connection

### What is SSH?

SSH (Secure Shell) is a network protocol that allows you to securely connect to and manage remote servers. It encrypts all communication between your local machine and the server, making it safe to use over the internet.

### Connecting to Your Server

To connect to your server via SSH, you'll need:

1. **Server IP address** - Provided by your hosting provider
2. **Username** - Usually `root` for initial access
3. **Password** - Provided by your hosting provider


> In the digital ocean setup, the root password is something you set when you first create the droplet

Open your terminal and use this command:

```bash
ssh root@YOUR_SERVER_IP
```

Replace `YOUR_SERVER_IP` with your actual server's IP address. For example:

```bash
ssh root@192.168.1.100
```

You'll be prompted for the root password. Type it carefully (it won't show on screen for security) and press Enter.

**Important:** If this is your first time connecting to this server, you'll see a message about the server's fingerprint. Type `yes` to continue.


## Step 2: Create a New Administrator User

### Why Not Use Root?

The `root` user has unlimited access to your entire system. If someone gains access to your root account, they can do anything - delete files, install malware, or steal data. It's also a username that malicious users know exists, so it's the first thing they will often check in case they strike it rich. It's a security best practice to create a regular user with administrative privileges and disable direct root login.

### Creating Your Admin User

Once connected as root, create a new user:

```bash
adduser yourusername
```

Replace `yourusername` with your preferred username (no spaces, lowercase recommended). You'll be prompted to:

1. Enter a password (choose a strong one!)
2. Confirm the password
3. Enter user information (optional - you can press Enter to skip)

### Grant Administrative Privileges

Add your new user to the `sudo` group to give them administrative privileges:

```bash
usermod -aG sudo yourusername
```

**Understanding the `-aG` flags:**
- `-a` (append): This flag tells `usermod` to **append** the user to the specified group(s) without removing them from any existing groups. Without this flag, the user would be removed from all other groups and only added to the `sudo` group.
- `-G` (groups): This flag specifies that you want to modify the user's **supplementary groups** (also called secondary groups). These are groups the user belongs to in addition to their primary group.

**Why use `-aG` instead of just `-G`?**
If you used only `-G sudo yourusername`, the user would be removed from all their current groups and only added to the `sudo` group. This could break access to other important groups they might already belong to. The `-a` flag ensures we're adding the user to the `sudo` group while preserving their existing group memberships.

### Test Your New User

Before logging out as root, test that your new user can use sudo:

```bash
su - yourusername
sudo whoami
```

You should see `root` as the output, confirming sudo access works.

## Step 3: Configure the Firewall (UFW)

### What is UFW?

UFW (Uncomplicated Firewall) is a user-friendly interface for managing iptables firewall rules on Ubuntu/Debian systems. It helps protect your server by controlling which network connections are allowed.

### Why Configure Firewall Before Enabling?

**Critical:** You must allow SSH connections through the firewall BEFORE enabling it. If you enable the firewall without allowing SSH, you'll be locked out of your server!

### Check UFW Status

First, check if UFW is already running:

```bash
sudo ufw status
```

If it shows "Status: inactive", you're good to proceed.

### Allow SSH Through Firewall

Allow SSH connections (port 22) through the firewall:

```bash
sudo ufw allow ssh
```

Or alternatively, you can specify the port directly:

```bash
sudo ufw allow 22
```

### Enable the Firewall

Now it's safe to enable UFW:

```bash
sudo ufw enable
```

You'll see a warning about disrupting existing connections. Type `y` and press Enter.

### Verify Firewall Status

Check that UFW is now active and SSH is allowed:

```bash
sudo ufw status
```

You should see something like:

```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
```

## Step 4: SSH Key Authentication

### What are SSH Keys?

SSH keys are a pair of cryptographic keys (public and private) that provide a more secure way to authenticate than passwords. The private key stays on your local machine, while the public key is placed on the server.

**Benefits of SSH keys:**
- More secure than passwords
- No need to type passwords repeatedly
- Can be used for automated scripts
- Resistant to brute force attacks

### Generate SSH Key Pair (On Your Local Machine)

**Important:** Do this on your local computer, not on the server!

#### On Windows (PowerShell or Command Prompt):

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

#### On macOS/Linux:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

**Understanding the `ssh-keygen` flags:**
- `-t rsa`: Specifies the **type** of key to generate. RSA is the most widely supported algorithm. Other options include `ed25519` (newer, more secure) or `ecdsa`.
- `-b 4096`: Sets the **bit length** of the key. 4096 bits provides strong security. The default is 2048 bits, but 4096 is recommended for better security.
- `-C "your_email@example.com"`: Adds a **comment** to the key. This is typically your email address and helps identify the key's owner. It's stored in the key file but doesn't affect security.

You'll be prompted for:

1. **File location**: Press Enter to use the default location (`~/.ssh/id_rsa`)
2. **Passphrase**: Enter a strong passphrase (recommended) or press Enter for no passphrase

> The passphrase ensures it's not just the ssh key someone needs, but the passphrase with it.

### Copy Public Key to Server

#### Method 1: Using ssh-copy-id (Recommended)

If you have `ssh-copy-id` available:

```bash
ssh-copy-id yourusername@YOUR_SERVER_IP
```

#### Method 2: Manual Copy

If `ssh-copy-id` isn't available, manually copy the key:

1. Display your public key:
```bash
cat ~/.ssh/id_rsa.pub
```

2. Copy the entire output (it's one long line)

3. On the server, create the SSH directory and authorized_keys file:
```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

4. Paste your public key into the file, save, and exit (Ctrl+X, then Y, then Enter)

5. Set proper permissions:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

> Another manual method of copying the key over is to use one of the file transfer programs that we've been using. Either the VSCode extension or CyberDuck.
>
> For CyberDuck, make sure to enable seeing hidden files and folders since the .ssh folder is considered hidden.

### Test SSH Key Authentication

From your local machine, try connecting with your new user:

```bash
ssh yourusername@YOUR_SERVER_IP
```

If everything is set up correctly, you should be logged in without entering a password.

### Disable Password Authentication

Once SSH keys are working, you can disable password authentication for added security:

1. Edit the SSH configuration:
```bash
sudo nano /etc/ssh/sshd_config
```

2. Find and modify these lines:
```
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

**Additional Security: Disable Root Login**
- `PermitRootLogin no`: Prevents direct root login via SSH, even with keys
- This forces all SSH access to go through your regular user account
- You can still use `sudo` to perform administrative tasks when needed

3. Restart the SSH service:
```bash
sudo systemctl restart ssh
```

**Warning:** Make sure your SSH key authentication is working before disabling password authentication!

### Change SSH Port (Optional but Recommended)

By default, SSH runs on port 22, which makes it a common target for automated attacks. Changing to a non-standard port can significantly reduce these attacks.

#### Step 1: Choose a New Port

Pick a port number between 1024 and 65535. Avoid well-known ports. Common choices:
- 2222, 2200, 2022 (easy to remember)
- 8022, 9022 (less predictable)

#### Step 2: Update SSH Configuration

1. Edit the SSH configuration:
```bash
sudo nano /etc/ssh/sshd_config
```

2. Find the line `#Port 22` and change it to:
```
Port 2222
```
(Replace 2222 with your chosen port number)

3. Save and exit the file

#### Step 3: Update Firewall Rules

**Critical:** Update your firewall before restarting SSH!

```bash
# Allow the new SSH port
sudo ufw allow 2222

# Remove the old SSH port rule (optional, but recommended)
sudo ufw delete allow 22
```

**Important Note:** If you originally used `ufw allow ssh` (which allows port 22), you'll need to update this rule because it's tied to the default SSH port. The `ssh` service name in UFW refers specifically to port 22, so when you change the SSH port, this rule no longer applies to your SSH service.

You can either:
- Use the specific port number: `sudo ufw allow 2222`
- Or update the SSH service rule: `sudo ufw delete allow ssh` then `sudo ufw allow 2222`

#### Step 4: Restart SSH Service

```bash
sudo systemctl restart ssh
```

**Important Safety Tip:** Keep your current terminal session open after restarting SSH. If something goes wrong with the new port configuration, you'll still have an active connection to troubleshoot and revert the changes if needed.

#### Step 5: Test the New Port

From your local machine, test the connection:
```bash
ssh -p 2222 yourusername@YOUR_SERVER_IP
```

If it works, you can update your SSH client configuration to use the new port by default.

#### Benefits of Changing SSH Port

- **Reduces automated attacks** - Bots typically scan port 22
- **Hides SSH service** - Makes your server less obvious to casual scanners
- **Simple security layer** - Easy to implement with minimal risk

#### Important Notes

- **Remember your new port** - Write it down somewhere safe
- **Update all SSH clients** - Any scripts or applications connecting to SSH need the new port
- **Keep port 22 rule until confirmed working** - Don't remove it until you've tested the new port
- **Some hosting providers** may have restrictions on which ports you can use

## Monitoring Server Access

### Why Monitor Access Attempts?

Even with proper security measures in place, it's important to monitor who is attempting to access your server. This helps you:
- Detect potential brute force attacks
- Identify unauthorized access attempts
- Monitor legitimate user activity
- Respond quickly to security threats

### Checking SSH Access Logs

The SSH service logs all connection attempts. Here are the most useful commands:

#### View Recent SSH Logs
```bash
sudo journalctl -u ssh -n 50
```
This shows the last 50 SSH-related log entries.

#### View Failed Login Attempts
```bash
sudo grep "Failed password" /var/log/auth.log
```
This shows all failed password attempts (useful for detecting brute force attacks).

#### View Successful Logins
```bash
sudo grep "Accepted" /var/log/auth.log
```
This shows all successful SSH logins with timestamps and source IPs.

#### View Current SSH Sessions
```bash
who
```
Shows who is currently logged into the server.

#### View Login History
```bash
last
```
Shows recent login history for all users.

### Understanding Log Entries

A typical SSH log entry looks like:
```
Dec 10 14:30:15 yourserver sshd[1234]: Accepted publickey for yourusername from 192.168.1.100 port 54321 ssh2
```

This tells you:
- **Timestamp**: When the login occurred
- **User**: Which user logged in
- **Method**: How they authenticated (publickey, password, etc.)
- **Source IP**: Where the connection came from
- **Port**: What port was used

### Monitoring for Suspicious Activity

Look for these warning signs:
- **Multiple failed attempts** from the same IP
- **Logins from unexpected locations** (different countries, unusual times)
- **Root login attempts** (should be disabled)
- **Password authentication attempts** (if you've disabled passwords)

### Setting Up Automated Monitoring (Optional)

For production servers, consider setting up automated monitoring:

```bash
# Install fail2ban to automatically block suspicious IPs
sudo apt update
sudo apt install fail2ban

# Configure fail2ban for SSH
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Fail2ban will automatically block IP addresses that show suspicious behavior (like multiple failed login attempts).

> Keep in mind, this includes you if you show suspicious behavior. For the sake of this class it might be best to leave this step out. Your server will be plenty secure by turning off password login and requiring the ssh key.

## Security Checklist

Before considering your server setup complete, verify:

- [ ] You can connect via SSH as your new user
- [ ] Your new user has sudo privileges
- [ ] UFW firewall is enabled and allows SSH
- [ ] SSH key authentication is working
- [ ] You've tested logging in with your new user account
- [ ] You know how to check server access logs

## What's Next?

Your server is now properly secured with:
- A non-root administrative user
- Firewall protection
- SSH key authentication

You're ready to proceed with setting up your development environment. The next step will be to install and configure Docker, which will allow you to run containerized applications and services on your server. Docker provides a consistent, isolated environment for your web applications and makes deployment much easier.

## Common Issues and Solutions

### "Permission denied (publickey)" Error
- Verify your public key is correctly copied to `~/.ssh/authorized_keys`
- Check file permissions (700 for .ssh directory, 600 for authorized_keys)
- Ensure you're using the correct username

### Locked Out of Server
- If you disabled password authentication too early, you might still be able to get into the server through the hosting provider. Digital ocean has a "console" that is effectively like logging into the server directly and thus would bypass any firewall issues. If it's a problem with the root password, there is a root password reset.

- Otherwise, it might be time to nuke the server and tell it to start from a blank slate again.

### Firewall Blocking Connections
- Check UFW status: `sudo ufw status`
- Temporarily disable if needed: `sudo ufw disable`
- Remember to re-enable after fixing the issue

Remember: Security is an ongoing process. Keep your system updated and monitor for any unusual activity!
