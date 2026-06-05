# Ansible — Kubernetes Cluster Setup

---

## Cluster overzicht

| Rol      | Hostname     | IP-adres       | OS             |FQDN      |
|----------|--------------|----------------|----------------|----------|
| Master   | k8s-master   | 192.168.1.50  | Ubuntu 22.04   |k8smaster.leerteam3.lab |
| Worker 1 | k8s-worker1  | 192.168.1.51  | Ubuntu 22.04   |k8smaster.leerteam3.lab |
| Worker 2 | k8s-worker2  | 192.168.1.52  | Ubuntu 22.04   |k8smaster.leerteam3.lab |
| NFS      | TrueNAS      | 192.168.1.20  | TrueNAS Scale  |truenas.leerteam3.lab |

---

## Vereisten

- Ansible geïnstalleerd op je lokale machine
- 3 Ubuntu 22.04 nodes bereikbaar via SSH
- TrueNAS met de volgende NFS exports beschikbaar:

| NFS export                    | Gebruikt door         |
|-------------------------------|-----------------------|
| `/mnt/pool/web-data`          | PVC `web-storage`     |
| `/mnt/pool/stream-data`       | PVC `stream-storage`  |
| `/mnt/pool/monitoring-data`   | PVC `monitoring-storage` |
| `/mnt/pool/audit-logs`        | PVC `audit-logs-storage` |

- Minimaal 2 vCPU's en 4 GB RAM per node

Ansible installeren:
```bash
sudo apt install ansible        # Linux
brew install ansible            # macOS
```

---

## Configuratie aanpassen

Alle instellingen staan in twee bestanden:

**`inventory.ini`** — IP-adressen en SSH-toegang:
```ini
[master]
k8s-master ansible_host=192.168.1.50

[workers]
k8s-worker1 ansible_host=192.168.1.51
k8s-worker2 ansible_host=192.168.1.52

[all:vars]
ansible_user=ubuntu
ansible_password=admin
ansible_become=true
ansible_become_method=sudo
ansible_become_password=admin
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

**`group_vars/all.yml`** — clusterinstellingen:
```yaml
cluster_name: homelab-cluster

master_ip: 192.168.1.50
worker_ips:
  - 192.168.1.51
  - 192.168.1.52

nfs_server: 192.168.1.20
nfs_storageclass_name: nfs-client

grafana_admin_password: "Admin1234!"

argocd_nodeport_http: 32080
argocd_nodeport_https: 32443

k8s_version: "1.32"
```

Pas deze waarden aan naar jouw omgeving voordat je het script uitvoert.

---

## Starten

### Verbinding testen voordat je begint
```bash
ansible all -i inventory.ini -m ping
```

### Volledige installatie (alles in één keer)
```bash
ansible-playbook -i inventory.ini site.yml
```

### Alleen een specifieke rol uitvoeren
```bash
# Alleen monitoring installeren
ansible-playbook -i inventory.ini site.yml --tags monitoring

# Alleen logging installeren
ansible-playbook -i inventory.ini site.yml --tags logging

# Alleen de applicatie deployen
ansible-playbook -i inventory.ini site.yml --tags applicatie
```

---

## Wat er geïnstalleerd wordt

### Installatiestappen (in volgorde)

| Stap | Onderdeel | Namespace | Role |
|------|-----------|-----------|------|
| 1 | `apt dist-upgrade`, `autoremove`, `autoclean` | — | `system` |
| 2 | Hostname, `/etc/hosts`, IPv6 uit, swap uit, kernel modules (`overlay`, `br_netfilter`), `nfs-common` | — | `common` |
| 3 | `containerd.io` via Docker repo, CRI plugin, SystemdCgroup, sandbox image `pause:3.9` | — | `containerd` |
| 4 | `kubelet`, `kubeadm`, `kubectl` v1.32 via pkgs.k8s.io | — | `kubernetes` |
| 5 | `kubeadm init`, kubeconfig, Calico CNI v3.27, join command genereren | `kube-system` | `master` |
| 6 | Workers automatisch joinen via gegenereerd join command | `kube-system` | `worker` |
| 7 | NGINX Ingress Controller, TLS-terminatie, routes `/web` + `/stream` | `ingress-nginx` | `ingress` |
| 8 | NFS CSI Driver (DaemonSet), StorageClass `nfs-client` (default) | `nfs-system` | `nfs` |
| 9 | Metrics Server + kube-prometheus-stack (Prometheus, Grafana, Alertmanager) | `kube-system` + `monitoring` | `monitoring` |
| 10 | Loki + Promtail (DaemonSet), PVC `audit-logs-storage` (NFS), Loki datasource in Grafana, RBAC + NetworkPolicies | `logging` | `logging` |
| 11 | ArgoCD via officiële manifests, NodePort 32080/32443 | `argocd` | `argocd` |
| 12 | Web Interface + Streaming & VOD Service, PVCs, HPA, RBAC + NetworkPolicies | `applicatie` | `applicatie` |

### Namespaces na installatie

| Namespace | Componenten |
|-----------|-------------|
| `kube-system` | kube-apiserver, etcd, kubelet, kube-proxy, kube-scheduler, kube-controller-manager, containerd, Calico CNI, Metrics Server |
| `ingress-nginx` | NGINX Ingress Controller |
| `nfs-system` | NFS CSI Driver (DaemonSet), StorageClass `nfs-client` |
| `applicatie` | Web Interface (Pod), Streaming & VOD Service (Pod), HPA, PVC `web-storage`, PVC `stream-storage`, ServiceAccount `web-sa`, NetworkPolicies |
| `monitoring` | Prometheus, Grafana (NodePort:32000), Alertmanager, PVC `monitoring-storage` |
| `logging` | Loki, Promtail (DaemonSet), PVC `audit-logs-storage`, ServiceAccount `promtail-sa`, ClusterRole, NetworkPolicies |
| `argocd` | ArgoCD server, repo-server, application-controller |

### Niet expliciet gevraagd — maar wel meegenomen

| Onderdeel | Waar | Waarom |
|-----------|------|--------|
| `apparmor` package installeren | `containerd` | Vereist op Ubuntu 22.04, anders blokkeert AppArmor de CRI socket |
| `disabled_plugins` CRI verwijderen uit containerd config | `containerd` | Ubuntu zet CRI standaard uit — dit breekt kubeadm |
| Sandbox image instellen op `pause:3.9` | `containerd` | Voorkomt dat containerd een incompatibele versie pakt |
| `--ignore-preflight-errors=NumCPU` | `master` | Voorkomt fout in lab-omgevingen met weinig CPU's |
| Join command automatisch doorgeven aan workers | `master` + `worker` | Workers joinen zonder handmatig kopiëren van het token |
| `nfs-common` installeren op alle nodes | `common` | Vereist op elke node om NFS-volumes te kunnen mounten |
| Kubernetes versie als variabele (`k8s_version`) | `kubernetes` + `group_vars` | Versie op één plek aanpasbaar; ingesteld op 1.32 |
| `helm status` check vóór elke Helm install | alle Helm roles | Script veilig opnieuw uitvoeren zonder fouten (idempotent) |
| Metrics Server `--kubelet-insecure-tls` patch | `monitoring` | Vereist in colocatie-omgeving zonder gesigneerde kubelet certs |
| Rolling Update Strategy op beide Deployments | `applicatie` | Geen downtime bij updates (US-05) |
| Self-healing via liveness/readiness probes | `applicatie` | Automatisch herstellen bij crash (US-04.2) |
| HPA op Web Interface (2–6 replicas) en Streaming (2–8 replicas) | `applicatie` | Autoscaling op basis van CPU (US-04.1, US-04.2) |
| `reclaimPolicy: Retain` op StorageClass | `nfs` | Voorkomt dat NFS-data verloren gaat bij PVC-verwijdering |
| Loki als datasource toevoegen aan Grafana via API | `logging` | Logs direct zichtbaar in Grafana zonder handmatige configuratie |
| RBAC: `ServiceAccount web-sa` + `Role web-reader` + `RoleBinding` | `applicatie` | Pods draaien met minimale rechten (least privilege) |
| RBAC: `ServiceAccount promtail-sa` + `ClusterRole promtail-logs-reader` + `ClusterRoleBinding` | `logging` | Promtail heeft cluster-breed leesrecht op pods/nodes/namespaces |
| Calico NetworkPolicy `default-deny-all` in `applicatie` en `logging` | beide namespaces | Alle ongeautoriseerde pod-communicatie geblokkeerd by default |
| NetworkPolicy `allow-from-ingress-nginx` | `applicatie` | Alleen NGINX Ingress mag applicatie-pods bereiken op poort 80 |
| NetworkPolicy `allow-loki-ingress` | `logging` | Loki accepteert alleen verkeer van Promtail en Grafana (`monitoring`) |
| NetworkPolicy `allow-promtail-egress` | `logging` | Promtail mag alleen naar Loki (3100) en DNS communiceren |

---

## Bereikbaarheid na installatie

| Service    | URL                                  | Gebruikersnaam | Wachtwoord |
|------------|--------------------------------------|----------------|------------|
| Grafana    | `http://192.168.1.51:32000`         | `admin`        | waarde van `grafana_admin_password` |
| Prometheus | `http://192.168.1.51:<nodeport>`    | —              | — |
| ArgoCD     | `https://192.168.1.50:32443`        | `admin`        | zie ArgoCD sectie hieronder |
| Web Interface | `https://<Ingress-IP>/web`        | —              | — |
| Streaming & VOD | `https://<Ingress-IP>/stream`  | —              | — |

Ingress IP opvragen:
```bash
kubectl get svc -n ingress-nginx
```

Prometheus NodePort opvragen:
```bash
kubectl get svc -n monitoring | grep prometheus
```

---

## Na de installatie verifiëren

```bash
# Alle nodes moeten Ready zijn
kubectl get nodes -o wide

# Alle pods moeten Running zijn (per namespace)
kubectl get pods -n kube-system
kubectl get pods -n ingress-nginx
kubectl get pods -n nfs-system
kubectl get pods -n applicatie
kubectl get pods -n monitoring
kubectl get pods -n logging
kubectl get pods -n argocd

# NFS StorageClass controleren (nfs-client moet default zijn)
kubectl get storageclass

# PVCs controleren (alle moeten Bound zijn)
kubectl get pvc -A

# HPA controleren
kubectl get hpa -n applicatie

# Ingress routes controleren
kubectl get ingress -n applicatie

# RBAC controleren
kubectl get serviceaccount -n applicatie
kubectl get role,rolebinding -n applicatie
kubectl get clusterrole,clusterrolebinding | grep promtail

# NetworkPolicies controleren
kubectl get networkpolicy -n applicatie
kubectl get networkpolicy -n logging
```

---

## ArgoCD

ArgoCD wordt geïnstalleerd via de officiële upstream manifests in de `argocd` namespace.  
ArgoCD haalt automatisch manifests op via Git Pull/Sync over HTTPS (GitHub CI/CD).

**Toegang na installatie:**
- UI: `https://192.168.1.50:32443`
- Gebruiker: `admin`
- Wachtwoord: wordt automatisch getoond aan het einde van de Ansible run

Wachtwoord handmatig opvragen:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

**NodePorts:**
| Poort | Doel  |
|-------|-------|
| 32080 | HTTP  |
| 32443 | HTTPS |

---

## Projectstructuur

```
Ansible-K8S/
├── inventory.ini          # IP-adressen en SSH-instellingen
├── site.yml               # Hoofdplaybook — volgorde van installatie
├── group_vars/
│   └── all.yml            # Alle variabelen (IP's, wachtwoorden, versies, namespaces)
└── roles/
    ├── system/            # apt update + upgrade
    ├── common/            # hostname, hosts, IPv6, swap, kernel modules, nfs-common
    ├── containerd/        # container runtime (containerd.io via Docker repo)
    ├── kubernetes/        # kubeadm, kubelet, kubectl v1.32
    ├── master/            # cluster init, kubeconfig, Calico CNI v3.27
    ├── worker/            # cluster join
    ├── ingress/           # NGINX Ingress Controller (namespace: ingress-nginx)
    ├── nfs/               # NFS CSI Driver + StorageClass nfs-client (namespace: nfs-system)
    ├── monitoring/        # Metrics Server + kube-prometheus-stack (namespace: monitoring)
    ├── logging/           # Loki + Promtail (namespace: logging)
    ├── argocd/            # ArgoCD GitOps CD (namespace: argocd)
    └── applicatie/        # Web Interface + Streaming & VOD Service (namespace: applicatie)
```

Calvin Wang
Stan Beukers
Christiaan van 't slot
Vinay Anroedh
