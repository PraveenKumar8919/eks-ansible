# eks-ansible

Ansible playbooks to deploy the observability stack and Roboshop application onto an EKS cluster using Helm.

This repo handles **application deployments only**. The cluster infrastructure is provisioned by [eks-infra](https://github.com/PraveenKumar8919/eks-infra).

---

## What this repo deploys

```
Kubernetes Cluster
├── kube-system namespace
│   ├── AWS Load Balancer Controller  ← creates ALB from Ingress resources
│   └── ExternalDNS                   ← auto-creates Route 53 records from Ingress
├── loki-live namespace
│   └── Loki              ← log aggregation, stores logs in S3 (via IRSA)
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

ALB (single load balancer, shared across all subdomains)
  ├── grafana.devopswithpraveen.online     → Grafana
  ├── prometheus.devopswithpraveen.online  → Prometheus
  └── loki.devopswithpraveen.online        → Loki

ExternalDNS
  └── reads Ingress annotations → creates Route 53 A records automatically

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
├── .github/workflows/
│   ├── config.yml                           # GitHub Actions — ansible-playbook on push
│   └── security-scan.yml                    # Trivy secret + IaC misconfiguration scan
├── ansible.cfg                              # Ansible configuration
├── group_vars/
│   ├── all.yml                              # Variables (namespaces, image tags, etc.)
│   ├── vault.yml                            # Ansible Vault — encrypted secrets (safe to commit)
│   └── vault.yml.example                    # Template for creating vault.yml
├── playbooks/
│   ├── observability.yml                    # Deploys: ALB → ExternalDNS → Loki → Prometheus → Alloy
│   └── roboshop.yml                         # Deploys: Roboshop app
└── roles/
    ├── aws-load-balancer-controller/        # Helm install ALB controller
    ├── external-dns/                        # Helm install ExternalDNS
    ├── loki/tasks/main.yml                  # Helm install Loki + Ingress
    ├── prometheus/tasks/main.yml            # Helm install kube-prometheus-stack + Ingress
    ├── alloy/
    │   ├── tasks/main.yml                   # Helm install Alloy
    │   └── files/alloy-config.alloy         # Alloy pipeline config (River format)
    └── roboshop/
        ├── tasks/main.yml                   # Helm install Roboshop
        └── files/helm/roboshop/             # Local Helm chart for Roboshop
```

---

## Secrets management (Ansible Vault)

All passwords are stored in `group_vars/vault.yml`, encrypted with Ansible Vault AES256. The file is safe to commit — nobody can read it without the vault password.

To create your own vault file:
```bash
cp group_vars/vault.yml.example group_vars/vault.yml.plain
# Edit vault.yml.plain with your passwords
ansible-vault encrypt group_vars/vault.yml.plain --output group_vars/vault.yml
rm group_vars/vault.yml.plain
```

The vault password is stored as the GitHub secret `ANSIBLE_VAULT_PASSWORD`.

---

## Local usage

**1. Clone and configure kubectl**
```bash
git clone https://github.com/PraveenKumar8919/eks-ansible.git
cd eks-ansible

# Point kubectl at your cluster (run after terraform apply in eks-infra)
aws eks update-kubeconfig --name eks-test-cluster --region us-east-1
```

**2. Set environment variables from eks-infra outputs**
```bash
# From the eks-infra directory
export LOKI_IAM_ROLE_ARN=$(terraform output -raw loki_iam_role_arn)
export LOKI_S3_BUCKET=$(terraform output -raw loki_s3_bucket)
export ACM_CERT_ARN=$(terraform output -raw acm_certificate_arn)
export ALB_CONTROLLER_ROLE_ARN=$(terraform output -raw alb_controller_iam_role_arn)
export EXTERNALDNS_ROLE_ARN=$(terraform output -raw externaldns_iam_role_arn)
export VPC_ID=$(terraform output -raw vpc_id)
```

**3. Deploy the observability stack**
```bash
ansible-playbook playbooks/observability.yml \
  --vault-password-file /path/to/.vault_pass \
  -e "loki_iam_role_arn=$LOKI_IAM_ROLE_ARN"
```

**4. Deploy Roboshop**
```bash
ansible-playbook playbooks/roboshop.yml \
  --vault-password-file /path/to/.vault_pass
```

**5. Access the stack**

| Service | URL |
|---------|-----|
| Grafana | https://grafana.devopswithpraveen.online |
| Prometheus | https://prometheus.devopswithpraveen.online |
| Loki | https://loki.devopswithpraveen.online |

---

## GitHub Actions (CI/CD)

The workflow at `.github/workflows/config.yml` is triggered manually from **Actions → EKS Configuration → Run workflow**. Run it after `eks-infra` has been applied and the cluster is up.

```
Push to main
     ↓
Configure AWS credentials
     ↓
Read Terraform outputs from eks-infra S3 state
     ↓
ansible-playbook observability.yml
  → helm install aws-load-balancer-controller  (namespace: kube-system)
  → helm install external-dns                  (namespace: kube-system)
  → helm install loki                          (namespace: loki-live)
  → helm install kube-prometheus-stack         (namespace: grafana)
  → helm install alloy                         (namespace: monitoring)
     ↓
ansible-playbook roboshop.yml
  → helm install roboshop                      (namespace: roboshop)
     ↓
Print access URLs
```

### Required GitHub secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM user secret key |
| `TF_STATE_BUCKET` | S3 bucket where eks-infra Terraform state is stored |
| `ANSIBLE_VAULT_PASSWORD` | Password to decrypt group_vars/vault.yml |

---

## Grafana access

After deployment, Grafana is available at:

**https://grafana.devopswithpraveen.online**

Login with:
- **Username:** `admin`
- **Password:** value of `vault_grafana_admin_password` in your vault

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
