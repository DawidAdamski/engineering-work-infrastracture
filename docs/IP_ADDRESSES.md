# Przypisane Adresy IP

## Pula MetalLB

**Zakres IP:** `192.168.10.200` - `192.168.10.210` (11 adresów)

## Przypisane Adresy

| Adres IP | Serwis | Namespace | Porty | Opis |
|:--|:--|:--|:--|:--|
| `192.168.10.200` | `qdrant` | `crm-rfm` | `6333` (REST), `6334` (gRPC) | Qdrant vector database |
| `192.168.10.201` | `argocd-server` | `argocd` | `80` (HTTP), `443` (HTTPS) | ArgoCD GitOps server |
| `192.168.10.202` | `n8n` | `crm-rfm` | `5678` (HTTP) | n8n workflow automation |
| `192.168.10.203` | `crm-api` | `crm-rfm` | `8000` (HTTP) | CRM API backend |

## Wolne Adresy

| Adres IP | Status | Możliwe zastosowanie |
|:--|:--|:--|
| `192.168.10.204` | ✅ Wolny | Dostępny dla nowych serwisów |
| `192.168.10.205` | ✅ Wolny | Dostępny dla nowych serwisów |
| `192.168.10.206` | ✅ Wolny | Dostępny dla nowych serwisów |
| `192.168.10.207` | ✅ Wolny | Dostępny dla nowych serwisów |
| `192.168.10.208` | ✅ Wolny | Dostępny dla nowych serwisów |
| `192.168.10.209` | ✅ Wolny | Dostępny dla nowych serwisów |
| `192.168.10.210` | ✅ Wolny | Dostępny dla nowych serwisów |

## Dostęp do Serwisów

### Qdrant
```bash
# REST API
curl http://192.168.10.200:6333/collections

# gRPC
# Connect to 192.168.10.200:6334
```

### ArgoCD
```bash
# HTTPS (zalecane)
https://192.168.10.201

# HTTP (jeśli włączone)
http://192.168.10.201
```

### n8n
```bash
# Web UI
http://192.168.10.202:5678
```

### CRM API
```bash
# Health check
curl http://192.168.10.203:8000/health

# API endpoints
curl http://192.168.10.203:8000/api/customers
```

## Konfiguracja

Adresy IP są przypisane statycznie w manifestach Kubernetes poprzez adnotację:
```yaml
annotations:
  metallb.universe.tf/loadBalancerIPs: 192.168.10.XXX
```

## Zmiana Przypisanego Adresu

Aby zmienić przypisany adres IP dla serwisu:

1. Edytuj odpowiedni plik w `k8s/`:
   - `qdrant.yaml` - Qdrant
   - `n8n.yaml` - n8n
   - `crm-api.yaml` - CRM API
   - `argocd-loadbalancer.yaml` - ArgoCD

2. Zmień wartość w adnotacji `metallb.universe.tf/loadBalancerIPs`

3. Zcommitować zmiany do Git - ArgoCD automatycznie zsynchronizuje

## Rozszerzenie Puli IP

Jeśli potrzebujesz więcej adresów, edytuj `k8s/metallb-config.yaml`:

```yaml
spec:
  addresses:
    - 192.168.10.200-192.168.10.250  # Rozszerzona pula (51 adresów)
```

**Uwaga:** Upewnij się, że rozszerzona pula nie koliduje z innymi urządzeniami w sieci.
