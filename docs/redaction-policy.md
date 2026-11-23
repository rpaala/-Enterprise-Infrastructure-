# Redaction Policy

This repository is public-facing and linked from a résumé. Nothing in it should help someone map, reach, or impersonate the real network.

## Never commit

- IP addresses and subnets (IPv4 or IPv6), including private ranges
- Hostnames, local DNS names and domains
- MAC addresses, disk or device serial numbers, Wi-Fi network names (SSIDs)
- Usernames, email addresses, account IDs
- Passwords, password hashes, API tokens, SSH or WireGuard private keys, kubeconfigs, k3s join tokens, age/SOPS keys
- Terraform state, `.env` files, firewall exceptions tied to specific devices
- Unblurred screenshots of admin UIs

## Use placeholders instead

| Real thing | Placeholder |
|---|---|
| Subnet | `<MGMT_SUBNET>`, `<LAB_SUBNET>`, or a pattern like `10.0.X.0/24` |
| Host address | `<PROXMOX_IP>`, `<NAS_IP>`, `<K3S_API_VIP>` |
| Hostname | Role names: `proxmox-01`, `k3s-cp-1`, `ops`, `nas-01` |
| Domain | `<LOCAL_DOMAIN>` or `lab.example` (reserved for documentation) |
| Example IPs in prose | Documentation ranges `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24` |
| Serial / MAC | `<DISK_SERIAL>`, `<MAC_ADDRESS>` |
| Secrets | `<API_TOKEN>`, or reference the secret store (`Bitwarden: proxmox-terraform-token`) |

Real values live in a password manager, a local `.env` (git-ignored), or encrypted files (SOPS/Ansible Vault) in a private repo.

## Before every commit

1. Run `gitleaks detect --no-git -v` (or the pre-commit hook). This repo's `.gitleaks.toml` adds rules for private IPs, MAC addresses and local domains.
2. Search your diff: `git diff --cached | grep -E '([0-9]{1,3}\.){3}[0-9]{1,3}'`.
3. Blur screenshots.

CI runs gitleaks on every push and pull request; a finding fails the check.

## If something leaks

Removing it in a new commit is not enough, because Git history keeps it. Rotate any secret immediately, then rewrite history (`git filter-repo`) and force-push, or recreate the repository. Treat any leaked secret as compromised.
