# blog-bind-doh

Companion code for the blog post
[Implementing DNS over HTTPS (DoH) with BIND and nginx](https://blog.bsdserver.nl/implementing-dns-over-https-doh-with-bind-and-nginx/).

It builds the post's "dedicated DoH proxy" setup with the complete production nginx configuration:

```
Client --HTTPS--> nginx :443 --HTTP--> doh-server :8054 --DNS--> BIND :53 (localhost)
```

- BIND as a resolver that only answers on localhost
- doh-server (m13253/dns-over-https) v2.3.10, built with `go install`, with the post's config and systemd unit
- nginx with the complete production configuration: TLS 1.2/1.3, HTTP/2, rate and connection limits,
  access control, request validation (GET, or POST with `application/dns-message`, otherwise 415),
  `/health`, and 404 for everything else
- a self-signed certificate by default, or your own (for example Let's Encrypt) certificate files

doh-server listens on port 8054, so it does not clash with the BIND statistics channel on 8053 from the
monitoring post in the same series.

## Requirements

- Terraform 1.6+ and a Proxmox VE cluster with an API token (only if you want Terraform to create the VM)
- Ansible core 2.16+ on your workstation
- A target running Ubuntu 24.04 (tested)
- Outbound DNS and HTTPS from the server (for resolving, and for downloading Go modules during the build)

Ubuntu 24.04 ships Go 1.22, while doh-server v2.3.10 needs Go 1.23.5 or newer. The role keeps the default
`GOTOOLCHAIN=auto`, so `go install` downloads a newer Go toolchain for the build. The build cache
(about 500 MB) is removed afterwards.

## Quick start

```bash
# 1. Create the VM (optional, skip if you already have a server)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configure BIND, doh-server and nginx
cd ../ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

# 3. Run the post's checks (GET, POST, wrong content type, /health, curl --doh-url, kdig +https)
ansible-playbook ../tests/verify.yml
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and edit it.

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `doh_nginx_server_name` | `dns.example.com` | nginx `server_name` and the self-signed certificate name |
| `doh_nginx_allow` | loopback, 192.168.0.0/16, 10.0.0.0/8 | Networks allowed to use `/dns-query` |
| `doh_nginx_selfsigned` | `true` | Generate a self-signed certificate; set `false` to use your own files |
| `doh_nginx_ssl_certificate` / `doh_nginx_ssl_certificate_key` | `/etc/nginx/ssl/<name>.crt/.key` | Certificate and key paths |
| `doh_nginx_rate` / `doh_nginx_burst` / `doh_nginx_conn_limit` | `100r/s` / `200` / `50` | Rate and connection limits |
| `doh_server_port` | `8054` | doh-server listen port (127.0.0.1 and ::1) |
| `doh_server_version` | `v2.3.10` | doh-server release to build |
| `doh_server_upstream` | `udp:127.0.0.1:53` | Upstream resolver(s) |
| `doh_server_remove_build_cache` | `true` | Remove the Go build cache after building |

## Layout

- `terraform/` creates Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider) and outputs an Ansible inventory
- `ansible/roles/bind_install` installs BIND
- `ansible/roles/doh_resolver` configures BIND as a localhost resolver
- `ansible/roles/doh_server` builds and runs doh-server
- `ansible/roles/doh_nginx` configures nginx as the TLS terminator
- `tests/verify.yml` runs the functional checks from the post

## License

MIT
