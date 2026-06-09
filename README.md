# eks-ansible

Ansible playbooks to deploy the observability stack and Roboshop application onto an EKS cluster using Helm.

This repo handles **application deployments only**. The cluster infrastructure is provisioned by [eks-infra](https://github.com/PraveenKumar8919/eks-infra).

---

## What this repo deploys

```
Kubernetes Cluster
├── loki-live namespace
│   └── Loki              ← log aggregation, stores logs in S3
├── grafana namespace
│   ├── Prometheus        ← metrics collection and storage
│   ├── Grafana           ← dashboards (Prometheus + Loki pre-wired)
│   └── Alertmanager      ← alert routing
├── monitoring namespace
│   └── Alloy (DaemonSet) ← runs on every node, collects logs + metrics
└── roboshop namespace
    └── Roboshop          ← 10-microservice e-commerce app
```

---

## Observability architecture

```
EKS Nodes
  └── Alloy (DaemonSet on every node)
        ├── collects pod logs      → Loki  (loki-live ns)
        └── scrapes pod metrics    → Prometheus (grafana ns)

Grafana (LoadBalancer)
  ├── data source: Prometheus  ← metrics dashboards
  └── data source: Loki        ← log exploration (LogQL)

Loki
  └── stores logs in S3 (via IRSA — no AWS credentials needed)
```

### What is Grafana Alloy?
Alloy is Grafana's unified observability collector. It runs as a **DaemonSet** — one pod per node — and collects both logs and metrics from all pods on that node. It replaces separate tools like Promtail (logs) and Grafana Agent (metrics).

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) | Playbook runner |
| [Helm](https://helm.sh/docs/intro/install/) | Kubernetes package manager |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | Kubernetes CLI |
| [AWS CLI](https://aws.amazon.com/cli/) | kubeconfig setup |
| Python packages: `kubernetes` | Required by Ansible Helm module |
| Ansible collection: `kubernetes.core` | Ansible Helm module |

Install dependencies:
```bash
pip install ansible kubernetes
ansible-galaxy collection install kubernetes.core
```

---

## Repository structure

```
eks-ansible/
├── .github/workflows/config.yml        # GitHub Actions — runs ansible-playbook on push
├── ansible.cfg                          # Ansible configuration
├── group_vars/
│   └── all.yml                          # All variables (namespaces, image tags, etc.)
├── playbooks/
│   ├── observability.yml                # Deploys: Loki → Prometheus/Grafana → Alloy
│   └── roboshop.yml                     # Deploys: Roboshop app
└── roles/
    ├── loki/tasks/main.yml              # Helm install Loki
    ├── prometheus/tasks/main.yml        # Helm install kube-prometheus-stack
    ├── alloy/
    │   ├── tasks/main.yml               # Helm install Alloy
    │   └── files/alloy-config.alloy     # Alloy pipeline config (River format)
    └── roboshop/
        ├── tasks/main.yml               # Helm install Roboshop
        └── files/helm/roboshop/         # Local Helm chart for Roboshop
```

---

## Local usage

**1. Clone and configure kubectl**
```bash
git clone https://github.com/PraveenKumar8919/eks-ansible.git
cd eks-ansible

# Point kubectl at your cluster (run after terraform apply in eks-infra)
aws eks update-kubeconfig --name eks-test-cluster --region us-east-1
```

**2. Get the Loki IAM role ARN from Terraform outputs**
```bash
# From the eks-infra directory
terraform output loki_iam_role_arn
terraform output loki_s3_bucket
```

**3. Deploy the observability stack**
```bash
export GRAFANA_ADMIN_PASSWORD="your-grafana-password"

ansible-playbook playbooks/observability.yml \
  -e "loki_iam_role_arn=<arn-from-step-2>" \
  -e "loki_s3_bucket=<bucket-from-step-2>"
```

**4. Deploy Roboshop**
```bash
ansible-playbook playbooks/roboshop.yml
```

**5. Get access URLs**
```bash
# Grafana
kubectl get svc -n grafana kube-prometheus-stack-grafana

# Roboshop
kubectl get svc -n roboshop web
```

---

## GitHub Actions (CI/CD)

The workflow at `.github/workflows/config.yml` runs automatically on every push to `main`.

```
Push to main
     ↓
Update kubeconfig (aws eks update-kubeconfig)
     ↓
Read loki_iam_role_arn + loki_s3_bucket from eks-infra Terraform state (S3)
     ↓
ansible-playbook observability.yml
  → helm install loki            (namespace: loki-live)
  → helm install kube-prometheus-stack  (namespace: grafana)
  → helm install alloy           (namespace: monitoring)
     ↓
ansible-playbook roboshop.yml
  → helm install roboshop        (namespace: roboshop)
     ↓
Print Grafana and Roboshop URLs
```

### Required GitHub secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM user secret key |
| `TF_STATE_BUCKET` | S3 bucket where eks-infra Terraform state is stored |
| `GRAFANA_ADMIN_PASSWORD` | Password you want to set for Grafana admin login |

---

## Grafana access

After deployment, get the Grafana URL:
```bash
kubectl get svc -n grafana kube-prometheus-stack-grafana \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Login with:
- **Username:** `admin`
- **Password:** value of your `GRAFANA_ADMIN_PASSWORD` secret

Both **Prometheus** and **Loki** data sources are pre-configured — no manual setup needed in Grafana.

---

## Roboshop services

| Service | Language | Port |
|---------|----------|------|
| web | Nginx | 80 |
| catalogue | Node.js | 8080 |
| cart | Node.js | 8080 |
| user | Node.js | 8080 |
| payment | Python | 8080 |
| shipping | Java | 8080 |
| mongodb | MongoDB | 27017 |
| mysql | MySQL | 3306 |
| redis | Redis | 6379 |
| rabbitmq | RabbitMQ | 5672 |
