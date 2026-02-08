# Cloud-1 — Automated Deployment of WordPress

## 📌 Project overview

This project is part of **Cloud-1** at **42**.  
The objective is to deploy a complete WordPress infrastructure on a **remote cloud server**, using **Docker** and **full automation**, inspired by the Inception project.

The deployment is designed to be:
- automated
- reproducible
- secure
- persistent after reboot

---

## 🧱 Architecture

The infrastructure follows the rule **one process = one container**.

### Services

| Service | Description |
|------|------|
| **Nginx** | Reverse proxy and public entry point |
| **WordPress (PHP-FPM)** | WordPress application |
| **MariaDB** | SQL database |
| **phpMyAdmin** | Database administration interface |
| **Certbot** (optional) | TLS certificate management (Let’s Encrypt) |

All services communicate through **internal Docker networks**.

---

## 🔐 Security

- Only ports **22 (SSH)**, **80 (HTTP)** and **443 (HTTPS)** are exposed
- The database is **not accessible from the Internet**
- A firewall (**UFW**) is enabled on the server
- Containers are isolated using Docker networks

---

## 💾 Data persistence

Persistent Docker volumes are used to ensure that data is not lost after:
- container restart
- `docker compose down`
- server reboot

### Volumes

- `db_data` → MariaDB data
- `wp_data` → WordPress files (uploads, plugins, themes)

---

## 🔁 Automation

The deployment is fully automated using **Ansible**.

### Automated steps

- Install Docker and Docker Compose on a fresh Ubuntu server
- Create the project directory on the server
- Copy all project files to the server
- Start the Docker stack automatically

This allows the infrastructure to be deployed on **any compatible server** with a single command.

---

## 🚀 Deployment

### Requirements

- Ubuntu server (cloud VM)
- SSH access using a private key
- Python installed on the target server

### Inventory example

```ini
[cloud1]
SERVER_IP ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/cloud1-key.pem
# cloud-1
