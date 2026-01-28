# A Boring Bastion That Works:

#### Expose Private Databases to SaaS Safely – Without Complex VPNs or Expensive Solutions

## The Actual Problem

So, if you are still uncertain if this solution is for you, this section is devoted to that discussion. You’ve found that a software as a service company wants to hook into a database you manage. So why not just poke a hole in your firewall and give them access? You could even add a rule that makes it so only their IP address can connect to you.

The problem comes in when you consider that database software was not made to be exposed to the internet. As software ages, more and more vulnerabilities are found, and this is especially true with software meant to be protected from the internet. Even if you have a very long password on your database, exposing it directly increases your attack surface in ways that firewalls alone do not fully mitigate.

SSH, on the other hand, has the direct purpose of creating a secure remote session. This illustrates that SSH was designed to reduce exposure points. I once wondered why they never made it so you could connect over SSH and transfer files over that same connection, but then I realized that they were lessening exposure points. Database software does a lot: user management, stored procedures, remote execution, replication. Exposing it directly increases the attack surface and may expose unexpected vulnerabilities.

---

## Why the Bastion / Tunnel Model Helps

A solution like a bastion host not only confirms that the individual comes from the IP you expect, but also establishes their identity through cryptographic keys. Access requires a secure, encrypted tunnel to be created first. No traffic can pass without that tunnel, and the tunnel itself cannot be established without possession of the corresponding private key. In practice, this means access is both origin-restricted and identity-verified before a single packet reaches the database.

SSH also provides a narrowly exposed, well-understood service surface, along with straightforward logging and auditing, making it easier to see who connected, when they connected, and from where.

---

## Alternatives and Why They May Not Fit

Now, there are other options like creating a site-to-site tunnel. These can be reliable, but they are a permanent tunnel, which is overkill if all you are doing is making one sort of application connection. They can also end up offering fragile and intermittent connections if misconfigured.

Another option is a client VPN. These are great for letting one of your employees connect to your environment. However, for this purpose it is not so good. The first issue is that a client VPN tends to allow too much traffic. This single use case of connecting to a database doesn’t need RDP or file server access, but they will get all that and more with a client VPN.

If that’s not enough of a detractor, the client VPN tends to often be manual. It is not as friendly to automate a client VPN, especially if your client VPN has multi-factor authentication, which it should.

---

## Practical Implementation

### A Minimal Bastion Configuration Example

This section shows a minimal, production-tested configuration for exposing a private SQL database to an external SaaS application using an SSH tunnel through a bastion host.

---

### 1. Provision the Bastion Host

- Linux server (Ubuntu 24.04.3 LTS confirmed working)
    
- Placed in a DMZ or assigned a public IP
    
- OpenSSH installed
    

> Note: you can install OpenSSH during the Ubuntu installation: `[x] Install OpenSSH server`

Restrict inbound SSH to the vendor’s external IP only and your own internal IP for management. It can be a good idea to test with your own external machine as well:

```
sudo ufw allow from <mgmt-ip-range>/27 to any port 22
sudo ufw allow from <external-user-ip> to any port 22 sudo ufw enable
```

It isn't a terrible idea to use a different port for SSH like `49222` rather than `22`. A higher port is less likely to be scanned and noticed by bad actors looking for access.

---

### 2. Create a Dedicated Access User

Create a user specifically for the external application. This user will have no password and no interactive shell access:

```
sudo adduser --disabled-password sqlaccess
```

---

### 3. Lock Down SSH Server Configuration

Edit `/etc/ssh/sshd_config` and disable unnecessary authentication methods:

#### Global SSH policy

These are editable fields, so find them to uncomment and change values:

`PermitRootLogin no 
`PasswordAuthentication no 
`KbdInteractiveAuthentication no 
`UsePAM yes 
`# optional: Port 49222

This actually just needs added:

```
AllowUsers bastionadmin sqlaccess 
Match User bastionadmin Address <mgmt-ip-range>/27     
	AllowUsers bastionadmin 
Match User sqlaccess Address <vendor-ip>,<vendor-ip2>     
	AllowUsers sqlaccess
```
Reboot the server:

```
sudo reboot
```
---

### 4. Create the directory structure needed for the vendor to connect

Create the SSH directory and add the provided public key:


```
sudo mkdir -p /home/sqlaccess/.ssh 
sudo vim /home/sqlaccess/.ssh/authorized_keys
```

Set correct ownership and permissions:

```
sudo chown -R sqlaccess:sqlaccess /home/sqlaccess/.ssh 
sudo chmod 700 /home/sqlaccess/.ssh 
sudo chmod 600 /home/sqlaccess/.ssh/authorized_keys
```

---

### 5. Restrict the Vendor’s SSH Key to Port Forwarding Only

OpenSSH allows you to attach restrictions directly to an SSH public key via the `authorized_keys` file.

This is the strongest place to enforce limitations because it applies even if `sshd_config` is later loosened.

**Important formatting note**

- `authorized_keys` **requires everything to be on a single line**
    
- Line breaks will cause the key to fail
    

```
command="echo 'Port forwarding only'; exec sleep infinity",no-pty,no-agent-forwarding,no-X11-forwarding,no-user-rc,
permitopen="<sql-server-ip>:1433" ssh-ed25519 AAAAC3...their-key
```

> Note: Some users may provide an `ssh-rsa` key instead of `ssh-ed25519`. The restrictions work the same.

---

### What Each Restriction Does

- **`command="echo 'Port forwarding only'; exec sleep infinity"`** — Overrides any attempted shell access; parks the user in a harmless infinite sleep.
    
- **`no-pty`** — Prevents allocation of a terminal; no interactive sessions, no shell tricks.
    
- **`no-agent-forwarding`** — Blocks SSH agent forwarding; prevents lateral movement using the user’s local keys.
    
- **`no-X11-forwarding`** — Disables X11 forwarding; eliminates GUI-based abuse paths.
    
- **`no-user-rc`** — Ignores the user’s `~/.ssh/rc` file; prevents execution of hidden startup commands.
    
- **`permitopen="<sql-server-ip>:1433"`** — Allows **only** this single TCP destination; all other port forwarding attempts are rejected.
    
- **`ssh-ed25519 AAAAC3...their-key`** — The user’s public SSH key; must appear **last**.
    

---

### 6. Allow Bastion Host to Reach the SQL Server

Verify that the bastion host can reach the database internally:


```
telnet <sql-server-ip> 1433
```

If this fails, be sure to check for firewall rules getting in the way.

---

### 7. External Vendor Tunnel Command

The SaaS provider establishes the tunnel using:

```
ssh -i path/to/private_key -N -L 1433:<sql-server-ip>:1433 sqlaccess@<bastion-ip> -p 49222
```

Their application then connects to:

`localhost:1433`

From the database’s perspective, the connection originates from the bastion host only.

---

### 8. Optional Monitoring and Abuse Protection

Install fail2ban to reduce noise and automated abuse:

`sudo apt install fail2ban`

Useful commands:

`# list of connections 

```
sudo journalctl -u ssh 
```

`# unban an IP address 

```
sudo fail2ban-client set sshd unbanip <ip-address>`
```
---

### Summary of Controls

| Feature                  | Purpose                              |
| ------------------------ | ------------------------------------ |
| Dedicated user           | Access isolation                     |
| Key-only login           | Prevents brute-force attacks         |
| No shell access          | Prevents misuse                      |
| `permitopen` restriction | Enforces single-port tunneling       |
| Firewall IP rules        | Reduces attack surface               |
| Bastion host             | Network segmentation and audit point |
