# Harbour Homelab — Common Role Documentation

Denne dokumentation beskriver **`common`-rollen** i Harbour Ansible setup, som bruges til at forberede alle noder (master og workers) inden k3s installation og yderligere services (Helm, Pi-hole, Nextcloud osv.).

---

## 📦 Formål

`common`-rollen har til formål at:

1. Installere **basale pakker**, som er nødvendige for k3s, Helm og generelle homelab-opgaver.
2. Opsætte **firewall (UFW)** med SSH adgang.
3. Forberede systemet til **k3s** ved at slå swap fra.
4. Give **logging og feedback** for hver vigtig opgave.

Rollen køres på **alle noder** før andre roller (k3s, monitoring, netværksstack).

---

## 🧩 Struktur

```
roles/common/
├─ defaults/
│  └─ main.yml      # Variabler
└─ tasks/
   └─ main.yml      # Hovedopgaver
```

---

### 1️⃣ Defaults

`roles/common/defaults/main.yml`

```yaml
ufw_enabled: true
ufw_ssh_port: 22
base_packages:
  - curl
  - ca-certificates
  - apt-transport-https
  - gnupg
  - software-properties-common
  - htop
  - vim
```

- `ufw_enabled` – kan slås til/fra efter behov
- `ufw_ssh_port` – port for SSH (standard 22)
- `base_packages` – pakker, der installeres på alle noder

---

### 2️⃣ Tasks

`roles/common/tasks/main.yml`

Hovedopgaver:

1. **APT cache opdatering**
   ```yaml
   - name: Update apt cache
   ```

2. **Installere base packages**
   ```yaml
   - name: Install base packages
   ```

3. **UFW installation og konfiguration**
   ```yaml
   - name: Install ufw
   - name: Allow SSH through UFW
   - name: Enable UFW
   ```

4. **Forberedelse til k3s**
   ```yaml
   - name: Ensure swap is off (required for k3s)
   - name: Comment out swap in fstab
   ```

5. **Logging**
   - Alle opgaver har `debug`-tasks, der viser om der er ændringer (`changed`) og hvad der er blevet udført.

---

## 🔹 Anvendelse

Eksempel playbook til alle noder:

```yaml
- hosts: all
  become: yes
  roles:
    - common
```

Dette sikrer, at:

- SSH adgang fungerer
- Firewall er aktiveret og korrekt konfigureret
- Systemet har alle nødvendige basale pakker
- Swap er deaktiveret, hvilket er **krav for Kubernetes/k3s**

---

## 💡 Fordele

- **Idempotent** – kan køres flere gange uden problemer.
- **Forbereder systemet til k3s/Helm**.
- **Centraliserede variabler** – nemt at ændre f.eks. SSH-port eller base packages.
- **Logging** – giver feedback på hvad der faktisk blev installeret og ændret.

---

## ⚓ Anbefalinger

- Kør `common` før **k3s-installation** på master og workers.
- Hold variabler opdaterede for at matche dit homelab setup.
- Tilføj evt. flere basale pakker eller ufw regler efter behov (f.eks. for MetalLB, Traefik eller andre services).  

> `common` role fungerer som fundamentet i Harbour – alt andet bygges ovenpå denne rolle.

