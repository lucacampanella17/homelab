# Homelab

Linux homelab on Debian 13 (UTM): SSH hardening, iptables firewall, Nginx reverse proxy, Docker, Prometheus and Grafana, automated backups.

## Structure
- nginx/ - reverse proxy configuration
- firewall/ - iptables rules
- monitoring/ - Prometheus and Grafana
- backup/ - backup scripts and strategy
- incidents/ - incident write-ups (symptom, cause, fix)

## Security rules
Nothing sensitive is ever committed to this repository:
- passwords, API keys, tokens, JWT secrets and `.env` files
- private keys and certificates
- real IP addresses, internal domains and personal data
Configurations are published sanitized, with placeholders such as `<PASSWORD>` or `<SERVER_IP>`.
