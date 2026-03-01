# Harbour – k3s Homelab Cluster ⚓

Harbour er mit lille **k3s-homelab cluster**, bygget på Lenovo M700-maskiner.  
Tanken er et simpelt, stabilt setup, der er let at automatisere med **Ansible**, og som kan udvides med services som MetalLB, Pi-hole/Unbound, Tailscale, storage m.m.

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
Worker-noderne håndterer workloads og kritiske services med HA.

---

## 📁 Repository-struktur

```text
theHarbour_ansible/
├─ collections/
   └─ requirements.yml
├─ Inventory/
│  ├─ group_vars/
│  │  └─ all.yml
│  └─ hosts.yml
├─ manifests/
│  └─ metallb.yaml
├─ playbooks/
│  ├─ breakdown.yml
│  ├─ build.yml
│  └─ first_run.yml
└─ roles/
   ├─ common/
   │  ├─ defaults/main.yml
   │  └─ tasks/main.yml
   └─ metallb/
      ├─ files/
      │  ├─ ipaddresspool.yml
      │  └─ l2advertisement.yml
      └─ tasks/main.yml
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

> Hvis du skal bruge Kubernetes-moduler (`k8s`) til MetalLB, installer pakkerne i pipx venv:

```bash
pipx inject ansible-core kubernetes openshift
```

---

### 4️⃣ Ansible inventory

**Fil:** `Inventory/hosts.yml`

```yaml
all:
  vars:
    ansible_user: pbech
    ansible_become_method: sudo
  children:
    k3s_master:
      hosts:
        harbour-lighthouse:
          ansible_host: 192.168.164.2
          ansible_become: true
    k3s_workers:
      hosts:
        harbour-dock-1:
          ansible_host: 192.168.164.3
          ansible_become: true
        harbour-dock-2:
          ansible_host: 192.168.164.4
          ansible_become: true
```

---

### 5️⃣ Test Ansible-forbindelse

```bash
ansible all -i Inventory/hosts.yml -m ping
```

Forventet output:

```text
harbour-dock-1 | SUCCESS => pong
harbour-dock-2 | SUCCESS => pong
harbour-lighthouse | SUCCESS => pong
```

---

### 6️⃣ Kør bootstrap & build

```bash
cd /mnt/c/Users/pibm9/Documents/theHarbour_ansible

# Bootstrap systemet
ansible-playbook -i Inventory/hosts.yml playbooks/first_run.yml --ask-pass

# Byg cluster + installér k3s, Helm og MetalLB
ANSIBLE_ROLES_PATH=./roles \
ansible-playbook -i Inventory/hosts.yml playbooks/build.yml --ask-pass \
  -e "ansible_become_password={YOUR_PASSWORD}"
```

Playbooken forventes at:

* installere basispakker, UFW, disable swap
* installere k3s server på `harbour-lighthouse`
* installere k3s agents på workers
* installere Helm
* deployere MetalLB med floating IP range

---

## ✅ Efter installation

```bash
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
```

Forventet output:

```text
harbour-lighthouse   Ready   control-plane
harbour-dock-1       Ready   worker
harbour-dock-2       Ready   worker
```

MetalLB vil allokere **ClusterIP / LoadBalancer IPs** fra range defineret i `manifests/metallb.yaml`.

---

## 🧱 Services i clusteret

### Pi-hole + Unbound

* Kører som **Deployment + ClusterIP service**
* DNS kan tilgås internt på `pihole.default.svc.cluster.local`
* Pi-hole administreres via `WEBPASSWORD` og `ADMINACCOUNT` (fra Kubernetes Secret)
* HA sikres via 2 replicas og worker-affinity

Se detaljeret dokumentation i [theHarbour Network repository](https://github.com/PeterBech/theHarbour_network)

### NFS Server (Persistent Storage)
* Kører på master (harbour-lighthouse) som **NFS-Server.**
* Eksporterer `/srv/k8s-storage` til LAN / k3s cluster (`192.168.164.0/24`).
* Worker-noder monterer NFS som **PersistentVolume** til workloads.
* Nem integration med StatefulSets og andre applikationer, der kræver delte volumener.
* Opsætning håndteres via Ansible-role ``nfs-server``.

Se detaljeret dokumentation i [NFS Server dokumentationen](Documents/NFS-Server.md)

### MetalLB

* Konfigurerer floating IP range på tværs af workers
* Kan eksponere LoadBalancer services til LAN/WAN
* Floating IP rutes til node med “mest overskud”

Læs mere om opsætning og detaljer i [MetalLB-dokumentationen](Documents/Metallb.md)

---

## 🗺️ Roadmap

1. k3s + Ansible bootstrap ✅
2. MetalLB ✅
3. Pi-hole + Unbound ✅
4. Tailscale
5. Monitoring (Prometheus / Grafana)
6. Persistent storage (Longhorn)
7. Nextcloud
8. Hardening, backups & dokumentation

---

> *Denne README fungerer som levende dokumentation og opdateres løbende.* ⚓

