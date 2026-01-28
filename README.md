# A Boring Bastion That Works — Ansible Playbook

> **Note:** The full Ansible playbook for automating this bastion setup is sold via Gumroad: [https://cwattersr.gumroad.com/l/eandqp](https://cwattersr.gumroad.com/l/eandqp)  
> This GitHub repository contains reference material and a manual installation guide. You can use it to understand the configuration or perform a manual setup.

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

> **Important:** The Ansible playbook is sold via Gumroad: [https://cwattersr.gumroad.com/l/eandqp](https://cwattersr.gumroad.com/l/eandqp)  
> This repo contains a manual install guide and reference materials only.

### 1. Set Variables

In your `group_vars` for the bastion host, set:

- vendor IPs  
- management IP range  
- bastion SSH port  
- database host + port  

### 2. Run the Playbook

If you have purchased the playbook from Gumroad, run it like any other Ansible playbook:

```
ansible-playbook -i inventory bastion.yml
```
## How the Vendor Connects

The vendor runs:

    ssh -i path/to/private_key -N -L 1433:<sql-server-ip>:1433 sqlaccess@<bastion-ip> -p 49222

Then their application connects to:

    localhost:1433

From the database’s perspective, all traffic originates from the bastion host.

---

## The Security Controls (In Plain English)

Feature                     | Why it matters
---------------------------- | -----------------------------------
Dedicated `sqlaccess` user   | Keeps vendor access isolated
Key-only login               | Prevents brute force attacks
No shell access              | Stops misuse
`permitopen`                 | Ensures only one tunnel destination
Firewall rules               | Reduces exposure
Bastion host                 | Provides audit and network segmentation

---

## Notes / Best Practices

- Place the bastion in a DMZ or assign it a public IP.
- Restrict SSH to the vendor IP and your management IP.
- Consider using a non-standard SSH port to reduce scanning noise.
- Assume someone may attempt brute-force attacks; plan accordingly.

---

## Important: Avoiding the “Missing sudo password” Issue

If your playbook fails with:

    TASK [bastion_ssh : Ensure OpenSSH server is installed] ... [ERROR]: Task failed: Missing sudo password

You need to allow your Ansible user to run sudo without a password.

Run:

    sudo visudo

Then add:

    ansible_user ALL=(ALL) NOPASSWD:ALL

It is recommended to use a **different user than `bastionadmin`** for the Ansible process.

---

## Support / Reference

For reference materials or manual install instructions, see this GitHub repository.  
To purchase the Ansible playbook, visit Gumroad: [https://cwattersr.gumroad.com/l/eandqp](https://cwattersr.gumroad.com/l/eandqp)
