# Harbour – k3s Homelab Cluster ⚓

Harbour er mit lille **k3s-homelab cluster**, bygget på Lenovo M700-maskiner.
Tanken er et simpelt, stabilt setup, der er let at automatisere med **Ansible**, og som kan udvides løbende med services som Ingress, cert-manager, storage m.m.

Clusteret er navngivet efter et havne-tema, hvor én node fungerer som fyrtårn (control-plane), og de øvrige som dokker, der håndterer lasten.

---

## 🧭 Overblik

* **Cluster-navn:** `harbour`
* **Kubernetes distribution:** k3s
* **Provisionering:** Ansible
* **Admin-miljø:** WSL (Ubuntu)

---

## ⚓ Node-navngivning & IP-plan

| Rolle         | Hostname             | IP-adresse      | Hardware                   | S/N      |
| ------------- | -------------------- | --------------- | -------------------------- | -------- |
| Control-plane | `harbour-lighthouse` | `192.168.164.2` | i3 · 4 GB RAM · 256 GB SSD |          |
| Worker        | `harbour-dock-1`     | `192.168.164.3` | i3 · 8 GB RAM · 128 GB SSD | S4BV7172 |
| Worker        | `harbour-dock-2`     | `192.168.164.4` | i3 · 8 GB RAM · 128 GB SSD | S4BA1778 |

`harbour-lighthouse` fungerer som master / control-plane og kører k3s-serveren.

---

## 📁 Repository-struktur

```text
ansible/
├─ collections/
│  └─ requirements.yml
├─ Inventory/
│  ├─ group_vars
│  │  └─ all.yml
│  └─ inventory.yml
├─ playbooks/
│  ├─ breakdown.yml
│  ├─ build.yml
│  └─ first_run.yml
└─ roles/
   └─ common/
      ├─ defaults/
      │  └─ main.yml
      └─ tasks/
         └─ main.yml
s
```

> Bemærk: `Inventory` har stort **I** (Linux er case-sensitive i WSL).

---

## 🔑 Første opsætning fra WSL (SSH + Ansible)

### 1️⃣ Generér SSH-nøgle

```bash
ssh-keygen -t ed25519 -C "harbour-homelab"
```

Tryk **Enter** for standard placering:

```
~/.ssh/id_ed25519
```

---

### 2️⃣ Kopiér SSH-nøglen til alle noder

```bash
ssh-copy-id pbech@192.168.164.2
ssh-copy-id pbech@192.168.164.3
ssh-copy-id pbech@192.168.164.4
```

Test adgang:

```bash
ssh pbech@harbour-lighthouse
```

---

### 3️⃣ Installér Ansible i WSL (pipx)

```bash
sudo apt update
sudo apt install -y pipx
pipx ensurepath
source ~/.bashrc
pipx install ansible-core
```

Verificér:

```bash
ansible --version
```

---

### 4️⃣ Ansible inventory

**Fil:** `ansible/Inventory/hosts.yml`

```yaml
all:
  children:
    k3s_master:
      hosts:
        harbour-lighthouse:
          ansible_host: 192.168.164.2
    k3s_workers:
      hosts:
        harbour-dock-1:
          ansible_host: 192.168.164.3
        harbour-dock-2:
          ansible_host: 192.168.164.4
  vars:
    ansible_user: pbech
```

---

### 5️⃣ Test Ansible-forbindelse

```bash
ansible all \
  -i "/mnt/c/Users/pibm9/Documents/K3s cluster/ansible/Inventory" \
  -m ping
```

Forventet output:

```text
harbour-dock-1 | SUCCESS => pong
harbour-dock-2 | SUCCESS => pong
harbour-lighthouse | SUCCESS => pong
```

---

### 6️⃣ Kør build-playbook

```bash
cd "/mnt/c/Users/pibm9/Documents/K3s cluster/ansible"
ansible-playbook -i Inventory playbooks/first_run.yml
# Hvis den beder om sudo-password, kør denne:
# ansible-playbook -i Inventory playbooks/build.yml --ask-pass -e "ansible_become_password={Password}"

# Hvis den ikke kan finde /common, så kør denne:
# ANSIBLE_ROLES_PATH=./roles ansible-playbook -i Inventory playbooks/first_run.yml --ask-pass -e "ansible_become_password={ Password }"
```

Playbooken forventes at:

* installere basispakker
* konfigurere systemet (UFW, swap off, logging)
* installere k3s server på `harbour-lighthouse`
* installere k3s agents på `harbour-dock-*`

---

## ✅ Efter installation

```bash
kubectl get nodes
```

Forventet output:

```text
harbour-lighthouse   Ready   control-plane
harbour-dock-1       Ready   worker
harbour-dock-2       Ready   worker
```

---

## 🧱 High Availability-strategi

Harbour er designet med **pragmatisk HA** i fokus.

* Single control-plane (`harbour-lighthouse`)
* HA workloads på worker-noder
* Ingen workloads på control-plane

**Principper**

* `replicas: 2` for kritiske services
* Pod anti-affinity mellem workers
* Replikeret storage (Longhorn, RF=2)

> Ægte HA control-plane kræver minimum 3 control-plane noder og er uden for nuværende scope.

---

## 🗺️ Roadmap

1. k3s + Ansible bootstrap ✅
2. MetalLB
3. Pi-hole + Unbound
4. Tailscale
5. Monitoring (Prometheus / Grafana)
6. Persistent storage (Longhorn)
7. Nextcloud
8. Hardening, backups & dokumentation

---

> *Denne README fungerer som levende dokumentation og opdateres løbende.* ⚓
