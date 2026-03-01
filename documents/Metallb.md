# MetalLB – LoadBalancer til Harbour k3s Cluster 🛰️

Denne guide beskriver, hvordan MetalLB er konfigureret i Harbour-homelab clusteret, og hvordan IP-pools og L2-advertisement fungerer.

---

## 📂 Namespace

MetalLB kører i eget namespace:

```bash
kubectl get ns
```

Forventet output:

```
metallb-system
```

---

## 🏗️ Deploy manifests

### IPAddressPool

Fil: `roles/metallb/files/ipaddresspool.yml`

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.164.10-192.168.164.19
```

* Denne range bruges af MetalLB til at allokere **LoadBalancer-IP’er** til services i clusteret.
* Ændr start/slut IP, hvis du udvider dit LAN.

---

### L2Advertisement

Fil: `roles/metallb/files/l2advertisement.yml`

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec: {}
```

* Layer2-announcement sørger for, at floating IP’erne annonceres på LAN’et.

---

## ⚙️ Deployment via Ansible

MetalLB deployes automatisk i `playbooks/build.yml`:

```yaml
- name: Install MetalLB
  hosts: localhost
  become: false
  roles:
    - metallb
```

Rollen `roles/metallb` gør følgende:

1. Installerer MetalLB manifests (`controller` + `speaker`) fra GitHub
2. Venter på, at controller-podden er kørende
3. Tjekker CRDs (IPAddressPool)
4. Anvender `IPAddressPool` og `L2Advertisement` fra rollefilerne

---

## 🔍 Test MetalLB

```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspools -A
kubectl get l2advertisements -n metallb-system
```

Forventet output:

```
NAME                          READY   STATUS    RESTARTS   AGE
controller-xxxxxx             1/1     Running   0          10m
speaker-xxxxxx                1/1     Running   0          10m

NAMESPACE        NAME           ADDRESSES
metallb-system   default-pool   192.168.164.10-192.168.164.19
```

* Nu kan du oprette services med `type: LoadBalancer` og få en IP fra poolen.

---

## 💡 Tips

* Opret flere IPAddressPools for forskellige services/netværk, hvis nødvendigt.
* Rollen er idempotent – gentag playbook for at opdatere IP ranges eller MetalLB-version.
* MetalLB versionen kan opdateres i `roles/metallb/tasks/main.yml` via manifest-URL.
