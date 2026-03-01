# Homelab DNS

3-node highly available DNS infrastructure using BIND9 and keepalived, fully managed with Ansible.

## Architecture

```
                  +-----------------+
                  |  dns1 (master)  |
                  |  192.168.22.21  |
                  +--------+--------+
                           |
              zone transfers (TSIG)
                     |           |
          +----------+--+   +---+-----------+
          | dns2 (sec.) |   | dns3 (sec.)   |
          | 192.168.22.39|   | 192.168.22.9 |
          +------+------+   +------+--------+
                 |                  |
                 +--- keepalived ---+
                     VIP: 192.168.22.10
```

- **dns1** — Master BIND9 server. Holds authoritative zone files.
- **dns2** — Secondary + keepalived BACKUP (priority 90)
- **dns3** — Secondary + keepalived MASTER (priority 100)
- **VIP 192.168.22.10** — Floating IP managed by keepalived. Clients should use this as their DNS resolver.

Zone transfers from master to secondaries are authenticated with a TSIG key. Keepalived monitors BIND9 health with `dig` checks every 5 seconds.

### DNS Zones

| Zone | Type |
|------|------|
| `rushdel.me` | Forward (A + CNAME records) |
| `home.rushdel.me` | Forward subzone (CNAME records) |
| `gitlab.rushdel.me` | Forward subzone (CNAME records) |
| `22.168.192.in-addr.arpa` | Reverse (auto-generated PTR records) |

## Prerequisites

- Ansible installed on your local machine
- SSH access as `server_admin` to all 3 DNS servers
- The Ansible Vault password (for TSIG key and keepalived secrets)

## Project Structure

```
.
├── ansible.cfg                          # Ansible config (inventory path, become settings)
├── inventory/
│   ├── hosts.yml                        # Server inventory and groups
│   └── group_vars/
│       ├── all/
│       │   ├── vars.yml                 # DNS records and all shared variables
│       │   └── vault.yml                # Encrypted secrets (TSIG key, keepalived password)
│       ├── dns_master.yml               # Master-specific vars (role=master, zone dir)
│       └── dns_secondary.yml            # Secondary-specific vars (role=secondary, zone dir)
├── playbooks/
│   ├── site.yml                         # Full infrastructure deploy
│   ├── dns-master.yml                   # Master-only deploy
│   ├── dns-secondary.yml                # Secondaries-only deploy
│   └── keepalived.yml                   # Keepalived-only deploy
└── roles/
    ├── bind9/                           # BIND9 install, config, zone files
    └── keepalived/                      # Keepalived install and config
```

## Initial Setup (Fresh Deploy)

1. Clone this repository.

2. Create the vault file with your secrets:
   ```bash
   ansible-vault create inventory/group_vars/all/vault.yml
   ```
   It must define:
   ```yaml
   vault_tsig_key_secret: "<your-tsig-key>"
   vault_keepalived_password: "<your-vrrp-password>"
   ```
   To generate a TSIG key: `tsig-keygen -a hmac-sha256 homelab-transfer-key`

3. Deploy the full infrastructure:
   ```bash
   ansible-playbook playbooks/site.yml --ask-vault-pass
   ```

4. Verify:
   ```bash
   dig @192.168.22.10 docker1.rushdel.me
   ```

## Managing DNS Records

All DNS records live in `inventory/group_vars/all/vars.yml`.

### Add / Change / Remove Records

1. Edit `inventory/group_vars/all/vars.yml` and modify the appropriate list:

   **A records** (`dns_a_records`):
   ```yaml
   - { name: myhost, ip: "192.168.22.50", comment: "My new server" }
   ```

   **CNAME records** (`dns_cname_records`):
   ```yaml
   - { name: myapp, target: "docker1", comment: "My app" }
   ```

   **Subzone CNAMEs** — use `dns_home_cname_records` or `dns_gitlab_cname_records`:
   ```yaml
   # home.rushdel.me subzone
   - { name: myservice, target: "docker2.rushdel.me.", comment: "My service" }
   ```
   Note: subzone targets must be FQDNs (ending with a dot).

   To remove a record, delete its line. To change one, update the `ip` or `target` value.

2. Deploy to the master:
   ```bash
   ansible-playbook playbooks/dns-master.yml --ask-vault-pass
   ```
   Secondaries will automatically pick up changes via zone transfer (the serial number increments on every deploy).

### Reverse DNS (PTR Records)

PTR records are auto-generated from `dns_a_records` for IPs in the `192.168.22.0/24` subnet. No manual management needed.

## Playbooks

| Playbook | What it does | When to use |
|----------|-------------|-------------|
| `site.yml` | Deploys BIND9 on all servers + keepalived on secondaries | Initial setup or full redeploy |
| `dns-master.yml` | Deploys BIND9 config and zones on the master only | After adding/changing DNS records |
| `dns-secondary.yml` | Deploys BIND9 config on secondaries only | After changing BIND9 options or adding a new secondary |
| `keepalived.yml` | Deploys keepalived on dns2 and dns3 | After changing keepalived/VIP config |

All playbooks require the vault password:
```bash
ansible-playbook playbooks/<playbook>.yml --ask-vault-pass
```

Use `--check` for a dry run:
```bash
ansible-playbook playbooks/dns-master.yml --ask-vault-pass --check
```

## Vault

Secrets are stored in `inventory/group_vars/all/vault.yml` (encrypted with Ansible Vault).

To view or edit secrets:
```bash
ansible-vault edit inventory/group_vars/all/vault.yml
```

Vault contains:
- `vault_tsig_key_secret` — TSIG shared secret for zone transfers
- `vault_keepalived_password` — VRRP authentication password

## Verification

Test DNS resolution through the VIP:
```bash
# Forward lookup
dig @192.168.22.10 docker1.rushdel.me

# Reverse lookup
dig @192.168.22.10 -x 192.168.22.29

# Query a subzone
dig @192.168.22.10 freshrss.home.rushdel.me

# Check SOA (useful for verifying serial after changes)
dig @192.168.22.10 rushdel.me SOA

# Query individual servers directly
dig @192.168.22.21 rushdel.me SOA   # master
dig @192.168.22.39 rushdel.me SOA   # dns2
dig @192.168.22.9  rushdel.me SOA   # dns3
```
