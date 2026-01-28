# A Boring Bastion That Works — Ansible Playbook

This Ansible playbook builds a minimal, production-tested SSH bastion that safely exposes a private SQL database to an external SaaS provider **without opening the database directly to the internet**.

It’s designed for simplicity, auditability, and security.

---

## Why This Exists

If a SaaS vendor needs access to your database, you do **not** want to poke a hole in your firewall and expose the database directly.

Database software was never meant to be internet-facing. Every exposed port increases the attack surface, even if the password is long and the network is “restricted.”

A bastion host fixes that by:

- enforcing IP restrictions
    
- requiring SSH key identity
    
- allowing only a single tunnel destination
    
- providing audit logs
    

---

## What This Playbook Does

When you run the playbook, it configures:

### Bastion Host Basics

- Installs OpenSSH
    
- Enforces key-based auth only
    
- Disables password login and interactive shell access
    
- Locks down SSH to only allow your management IP range and the vendor IP
    

### Vendor Access Controls

- Creates a dedicated `sqlaccess` user
    
- Creates `~/.ssh/authorized_keys`
    
- Adds the vendor’s public key
    
- Restricts that key to port forwarding only, with:
    
    - no shell
        
    - no PTY
        
    - no agent forwarding
        
    - no X11
        
    - only a single `permitopen` destination
        

### Optional Hardening

- Installs fail2ban
    

---

## Usage

### 1. Set Variables

In your `group_vars` for the bastion host, set:

- vendor IPs
    
- management IP range
    
- bastion SSH port
    
- database host + port
    

### 2. Run the Playbook

Run it like any other Ansible playbook:

```
ansible-playbook -i inventory bastion.yml
```

---

## How the Vendor Connects

The vendor runs:

```
ssh -i path/to/private_key -N -L 1433:<sql-server-ip>:1433 sqlaccess@<bastion-ip> -p 49222
```

Then their app connects to:

`localhost:1433`

From the database’s perspective, the traffic comes only from the bastion host.

---

## The Security Controls (In Plain English)

| Feature                    | Why it matters                      |
| -------------------------- | ----------------------------------- |
| Dedicated `sqlaccess` user | Keeps vendor access isolated        |
| Key-only login             | Prevents brute force                |
| No shell access            | Stops misuse                        |
| `permitopen`               | Ensures only one tunnel destination |
| Firewall rules             | Reduces exposure                    |
| Bastion host               | Provides audit and segmentation     |

---

## Notes / Best Practices

- Put the bastion in a DMZ or give it a public IP.
    
- Keep SSH restricted to the vendor IP and your management IP.
    
- Consider a non-standard SSH port to reduce random scanning noise.
    
- If you are exposing this to a vendor, assume someone will try to brute force it anyway.
    

---

## Important: Avoiding the “Missing sudo password” Issue

If your playbook fails with:

```
TASK [bastion_ssh : Ensure OpenSSH server is installed] ... [ERROR]: Task failed: Missing sudo password`
```

You need to allow your Ansible user to run sudo without a password.

Add the following to your sudoers file:

```
sudo visudo
```

Then add:

```
ansible_user ALL=(ALL) NOPASSWD:ALL
```

It is a good practice to use a **different user than `bastionadmin`** (or whatever user you use yourself) for the Ansible process.

