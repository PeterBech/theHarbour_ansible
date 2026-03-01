# NFS Server – k3s Homelab Cluster ⚓

Denne guide beskriver opsætning og konfiguration af **NFS-serveren** på Harbour-homelab clusterets master-node (`harbour-lighthouse`).
NFS bruges til at levere persistent storage til workloads, og klienterne (workers) kan mount'e NFS shares via `nfs-common`.

---

## 📁 Repository

**Role:** `roles/nfs-server/`

```text
nfs-server/
├─ defaults/
│  └─ main.yml        # Definerer exports og NFS-pakker
├─ handlers/
│  └─ main.yml        # Håndtering af NFS service restart
└─ tasks/
   └─ main.yml        # Installation, opsætning og aktivering af NFS
```

---

## ⚙️ NFS Role – variabler

**defaults/main.yml**

```yaml
nfs_exports:
  - path: /srv/k8s-storage
    network: 192.168.164.0/24
    options: rw,sync,no_subtree_check,no_root_squash

nfs_packages:
  - nfs-kernel-server
```

* `nfs_exports`: Liste over kataloger, der skal eksporteres via NFS, med tilhørende netværk og mount-options.
* `nfs_packages`: Pakker, der skal installeres for NFS-serveren.

---

## 🔧 Tasks

**tasks/main.yml** håndterer:

1. Installation af NFS-pakker (`nfs-kernel-server`)
2. Oprettelse af eksport-kataloger (`/srv/k8s-storage`)
3. Konfiguration af `/etc/exports`
4. Aktivering af exports (`exportfs -ra`)
5. Sikring af at NFS-serveren kører og er enabled

Eksempel på task:

```yaml
- name: Ensure export directories exist
  file:
    path: "{{ item.path }}"
    state: directory
    owner: nobody
    group: nogroup
    mode: "0777"
  loop: "{{ nfs_exports }}"
```

---

## 🔄 Handlers

**handlers/main.yml** genstarter NFS-serveren, hvis konfiguration ændres:

```yaml
- name: Restart NFS
  service:
    name: nfs-kernel-server
    state: restarted
```

---

## 🛠️ Integration i playbook

I `playbooks/build.yml` er NFS-serveren installeret og konfigureret på master:

```yaml
- name: Setup NFS server on master
  hosts: k3s_master
  become: true
  roles:
    - nfs-server
```

På worker-noder installeres NFS-klient:

```yaml
- name: Install NFS client utilities
  hosts: k3s_workers
  become: true
  tasks:
    - name: Install nfs-common
      apt:
        name: nfs-common
        state: present
        update_cache: true
```

---

## 🧪 Test af NFS mount

Fra en worker kan du teste mount:

```bash
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs 192.168.164.2:/srv/k8s-storage /mnt/test-nfs
ls /mnt/test-nfs
sudo umount /mnt/test-nfs
```

> Bemærk: Master IP er `192.168.164.2` i dette setup.

---

## ✅ Tips & fejlretning

* Hvis mount hænger, tjek firewall (UFW) på master:

```bash
sudo ufw allow from 192.168.164.0/24 to any port nfs
sudo ufw allow 111,2049/tcp
sudo ufw allow 111,2049/udp
```

* Sørg for at eksporterne er aktive:

```bash
sudo exportfs -v
```

* NFS serverens logs kan findes med:

```bash
journalctl -u nfs-kernel-server
```

---

> Denne dokumentation kan linkes direkte fra `README.md` som:
> `[NFS-Server opsætning](Documents/NFS-Server.md)`
