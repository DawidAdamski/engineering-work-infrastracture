# Project Links and References

This document contains all links and references used in the infrastructure project.

## 📚 Documentation Links

### Official Documentation

#### Kubernetes & Minikube
- **[Minikube Documentation](https://minikube.sigs.k8s.io/docs/)** - Minikube official documentation
- **[Minikube Persistent Volumes](https://minikube.sigs.k8s.io/docs/handbook/persistent_volumes/)** - Persistent volumes guide
- **[Kubernetes Documentation](https://kubernetes.io/docs/)** - Official Kubernetes documentation
- **[kubectl Installation](https://kubernetes.io/docs/tasks/tools/)** - kubectl CLI installation guide
- **[Helm Documentation](https://helm.sh/docs/)** - Helm package manager documentation

#### ArgoCD
- **[ArgoCD Releases](https://github.com/argoproj/argo-cd/releases)** - ArgoCD GitHub releases
- **[ArgoCD Documentation](https://argo-cd.readthedocs.io/)** - Official ArgoCD documentation
- **[ArgoCD Installation](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)** - ArgoCD installation manifest

#### PostgreSQL
- **[PostgreSQL Docker Official Image](https://www.docker.com/blog/how-to-use-the-postgres-docker-official-image/)** - Docker blog post on PostgreSQL image
- **[PostgreSQL Docker Hub](https://hub.docker.com/_/postgres)** - Official PostgreSQL images on Docker Hub
- **[PostgreSQL Docker Documentation](https://github.com/docker-library/docs/blob/master/postgres/README.md)** - PostgreSQL Docker image documentation
- **[DigitalOcean: Deploy Postgres to Kubernetes](https://www.digitalocean.com/community/tutorials/how-to-deploy-postgres-to-kubernetes-cluster)** - Tutorial on deploying PostgreSQL to Kubernetes

#### Qdrant
- **[Qdrant Docker Hub](https://hub.docker.com/r/qdrant/qdrant)** - Qdrant images on Docker Hub
- **[Qdrant Releases](https://github.com/qdrant/qdrant/releases)** - Qdrant GitHub releases
- **[Qdrant Installation Guide](https://qdrant.tech/documentation/guides/installation/)** - Qdrant installation documentation
- **[Qdrant Helm Charts](https://qdrant.to/helm)** - Qdrant Helm charts reference

#### n8n
- **[n8n Docker Hub](https://hub.docker.com/r/n8nio/n8n)** - n8n images on Docker Hub
- **[n8n Releases](https://github.com/n8n-io/n8n/releases)** - n8n GitHub releases
- **[n8n Documentation](https://docs.n8n.io/)** - Official n8n documentation

#### Docker & Container Images
- **[Docker Hub](https://hub.docker.com/)** - Primary container image repository
- **[Docker Documentation](https://docs.docker.com/)** - Docker official documentation
- **[Red Hat Container Catalog](https://catalog.redhat.com/software/containers/)** - Red Hat UBI and certified images
- **[Docker Hub - redhat/ubi10](https://hub.docker.com/r/redhat/ubi10)** - Red Hat UBI10 images

#### Technical References
- **[Linux Mount Propagation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)** - Linux kernel mount propagation documentation
- **[CRI-O Storage Documentation](https://github.com/cri-o/cri-o/blob/main/docs/crio.conf.5.md)** - CRI-O storage configuration
- **[Overlay Filesystem](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)** - Linux overlay filesystem documentation
- **[curl / jq](https://stedolan.github.io/jq/)** - jq JSON processor documentation

## 🔗 Project Repositories

### Source Code Repositories
- **[engineering-work-infrastructure](https://github.com/DawidAdamski/engineering-work-infrastracture)** - This infrastructure repository (GitOps source)
- **[engineering-work-crm](https://github.com/DawidAdamski/engineering-work-crm)** - CRM API application repository (separate repo)

## 📖 Tutorials and Guides

### PostgreSQL Deployment
- **[Medium: PostgreSQL HA in Kubernetes Minikube](https://medium.com/@kurangdoa/postgres-ha-in-kubernetes-minikube-7eb676884a2d)** - PostgreSQL High Availability setup guide

### Kubernetes Tutorials
- **[DigitalOcean Kubernetes Tutorials](https://www.digitalocean.com/community/tags/kubernetes)** - Kubernetes tutorials and guides

## 🐳 Container Image Registries

### Docker Hub
- **[Docker Hub Homepage](https://hub.docker.com/)** - Main container registry
- **[PostgreSQL Images](https://hub.docker.com/_/postgres)** - Official PostgreSQL images
- **[Qdrant Images](https://hub.docker.com/r/qdrant/qdrant)** - Qdrant vector database images
- **[n8n Images](https://hub.docker.com/r/n8nio/n8n)** - n8n workflow automation images

### Quay.io
- **[ArgoCD Images](https://quay.io/repository/argoproj/argocd)** - ArgoCD images on Quay.io
- **[ArgoCD Repo Server](https://quay.io/repository/argoproj/argocd-repo-server)** - ArgoCD Repo Server images
- **[ArgoCD Application Controller](https://quay.io/repository/argoproj/argocd-applicationset-controller)** - ArgoCD Application Controller images
- **[ArgoCD Dex](https://quay.io/repository/argoproj/argocd-dex-server)** - ArgoCD Dex server images

### Red Hat Container Registry
- **[Red Hat Container Catalog](https://catalog.redhat.com/software/containers/)** - Red Hat certified container images
- **[UBI10 Python 3.11](https://catalog.redhat.com/software/containers/ubi10/python-311)** - Red Hat UBI10 with Python 3.11

## 🔧 Tools and Utilities

### Command Line Tools
- **[kubectl](https://kubernetes.io/docs/tasks/tools/)** - Kubernetes command-line tool
- **[Helm](https://helm.sh/docs/)** - Kubernetes package manager
- **[Minikube](https://minikube.sigs.k8s.io/docs/)** - Local Kubernetes cluster
- **[Docker](https://docs.docker.com/)** - Container platform
- **[curl](https://curl.se/)** - Command-line HTTP client
- **[jq](https://stedolan.github.io/jq/)** - JSON processor

### Development Tools
- **[ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)** - ArgoCD command-line interface

## 📝 Internal Documentation

### Project Documentation Files
- `README.md` - Main project README
- `docs/README.architecture.md` - System architecture documentation
- `docs/README.argocd.md` - ArgoCD setup and usage
- `docs/README.crm-api.md` - CRM API deployment documentation
- `docs/README.embeddings-worker.md` - n8n embedding worker documentation
- `docs/README.postgresql.md` - PostgreSQL database documentation
- `docs/README.qdrant.md` - Qdrant vector database documentation
- `docs/README.deployment.md` - Deployment guide
- `docs/README.k8s-resources.md` - Kubernetes resources documentation

### Troubleshooting Documentation
- `k8s/PODMAN_NOTES.md` - CRI-O volume mount issue documentation
- `k8s/MOUNT_PROPAGATION_EXPLAINED.md` - Mount propagation technical explanation

## 🌐 Service URLs

### Internal Kubernetes Service URLs
- **PostgreSQL**: `postgres.crm-rfm.svc.cluster.local:5432`
- **Qdrant**: `qdrant.crm-rfm.svc.cluster.local:6333` (HTTP), `qdrant.crm-rfm.svc.cluster.local:6334` (gRPC)
- **n8n**: `n8n.crm-rfm.svc.cluster.local:5678`
- **CRM API**: `crm-api.crm-rfm.svc.cluster.local:8000`
- **ArgoCD Server**: `argocd-server.argocd.svc.cluster.local:443`

### External Access URLs (via Port-Forward)
- **ArgoCD UI**: `https://localhost:8080` (when port-forwarding)
- **CRM API**: `http://localhost:8000` (when port-forwarding)
- **n8n UI**: `http://localhost:5678` (when port-forwarding)
- **Qdrant**: `http://localhost:6333` (when port-forwarding)
- **PostgreSQL**: `localhost:5432` (when port-forwarding)

### Ingress URLs (if configured)
- **CRM API**: `http://crm.local` (requires Ingress and /etc/hosts entry)
- **n8n UI**: `http://crm.local/n8n` (if configured)

## 🔐 API Endpoints

### OpenAI API
- **Embeddings API**: `https://api.openai.com/v1/embeddings`
- **OpenAI Documentation**: `https://platform.openai.com/docs/`

### CRM API Endpoints (when deployed)
- **Health Check**: `GET /health`
- **List Customers**: `GET /api/customers`
- **Get Customer**: `GET /api/customers/{id}`
- **Create Customer**: `POST /api/customers`
- **RFM Analysis**: `GET /api/rfm`
- **Similar Customers**: `GET /api/similar/{id}`

### Qdrant API Endpoints
- **Collections**: `GET /collections`
- **Create Collection**: `PUT /collections/{name}`
- **Upsert Points**: `POST /collections/{name}/points`
- **Search Points**: `POST /collections/{name}/points/search`

## 📊 Version Information

### Current Versions
- **PostgreSQL**: `18-alpine`
- **Qdrant**: `v1.15.5`
- **n8n**: `1.118.1`
- **ArgoCD**: `v3.1.9`
- **Kubernetes**: `v1.30.0` (Minikube)
- **Minikube**: `≥ 1.33`
- **kubectl**: `≥ 1.30`

### Version Check Links
- **[PostgreSQL Tags](https://hub.docker.com/_/postgres/tags)** - Check available PostgreSQL versions
- **[Qdrant Tags](https://hub.docker.com/r/qdrant/qdrant/tags)** - Check available Qdrant versions
- **[n8n Tags](https://hub.docker.com/r/n8nio/n8n/tags)** - Check available n8n versions
- **[ArgoCD Releases](https://github.com/argoproj/argo-cd/releases)** - Check ArgoCD releases

## 📚 Additional Resources

### Kubernetes Learning
- **[Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)** - Kubernetes fundamentals
- **[Kubernetes Concepts](https://kubernetes.io/docs/concepts/)** - Core Kubernetes concepts

### GitOps
- **[GitOps Principles](https://www.gitops.tech/)** - GitOps methodology
- **[ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)** - ArgoCD best practices

### Container Security
- **[Container Security Best Practices](https://kubernetes.io/docs/concepts/security/pod-security-standards/)** - Kubernetes security guidelines
- **[Docker Security](https://docs.docker.com/engine/security/)** - Docker security documentation

---

**Last Updated**: 2025-11-03  
**Maintained by**: Engineering Thesis Infrastructure Project

