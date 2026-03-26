# ToggleMaster - GitOps (ArgoCD)

Repositório GitOps que define o estado desejado de todas as aplicações e addons do cluster Kubernetes. O ArgoCD monitora este repositório e sincroniza automaticamente as mudanças no AKS.

## Como funciona

```
Developer faz push ──► CI Pipeline ──► Build e Push no ACR
                                              │
                                              ▼
                                    Atualiza image tag
                                    no GitOps repo
                                              │
                                              ▼
                           ArgoCD detecta mudança ──► Sincroniza no AKS
```

1. O desenvolvedor faz push no repositório do serviço
2. A pipeline CI faz build da imagem, scan de segurança e push no ACR
3. O job `update-gitops` atualiza a tag da imagem no `kustomization.yaml` deste repositório
4. O ArgoCD detecta a mudança e aplica automaticamente no cluster AKS

## Estrutura do repositório

```
fiap-tc-3-gitops/
│
├── bootstrap/
│   └── bootstrap-app.yaml            # App-of-Apps: ponto de entrada do ArgoCD
│
├── applicationsets/
│   ├── kustomization.yaml             # Agrupa os ApplicationSets
│   ├── addons-app.yaml                # Application para addons do cluster
│   └── apps-appset.yaml               # ApplicationSet para microsserviços
│
├── addons/                            # Ferramentas do cluster
│   ├── kustomization.yaml             # Lista de addons
│   ├── ingress-nginx/                 # Ingress Controller
│   │   ├── application.yaml           # ArgoCD Application (Helm)
│   │   ├── values.yaml                # Configurações do Helm chart
│   │   └── config.json                # Configurações adicionais
│   └── sonarqube/                     # Análise de qualidade de código
│       ├── application.yaml           # ArgoCD Application (Helm) + Ingress App
│       ├── values.yaml                # Configurações do Helm chart
│       └── ingress.yaml               # Ingress para acesso externo
│
└── apps/                              # Microsserviços
    ├── auth/                          # Serviço de autenticação (Go)
    │   ├── base/
    │   │   ├── kustomization.yaml
    │   │   ├── deployment.yaml
    │   │   └── service.yaml
    │   └── overlays/prod/
    │       ├── kustomization.yaml
    │       ├── configmap.yaml
    │       ├── db-init-job.yaml
    │       ├── ingress.yaml
    │       └── secrets.yaml           # (gitignored)
    │
    ├── flags/                         # Serviço de feature flags (Python)
    │   ├── base/
    │   └── overlays/prod/
    │
    ├── targeting/                     # Serviço de targeting rules (Python)
    │   ├── base/
    │   └── overlays/prod/
    │
    ├── evaluation/                    # Serviço de avaliação de flags (Go)
    │   ├── base/
    │   └── overlays/prod/
    │
    ├── analytics/                     # Serviço de analytics (Python)
    │   ├── base/
    │   └── overlays/prod/
    │
    └── loadtest/                      # Teste de carga com k6
        ├── base/
        └── overlays/prod/
```

## Padrão de Bootstrap (App-of-Apps)

O ArgoCD usa o padrão **App-of-Apps** para gerenciar todos os recursos a partir de um único ponto de entrada.

```
bootstrap-app (aplicado manualmente uma única vez)
    │
    ├── cluster-addons (Application)
    │   ├── addon-ingress-nginx
    │   ├── addon-sonarqube
    │   └── addon-sonarqube-ingress
    │
    └── apps (ApplicationSet)
        ├── app-auth
        ├── app-flags
        ├── app-targeting
        ├── app-evaluation
        ├── app-analytics
        └── app-loadtest
```

### Aplicação inicial

Para iniciar todo o fluxo GitOps, aplique o bootstrap uma única vez:

```bash
kubectl apply -f bootstrap/bootstrap-app.yaml
```

A partir desse momento, o ArgoCD gerencia todo o resto automaticamente.

## Kustomize: Base e Overlays

Os microsserviços seguem o padrão **base/overlays** do Kustomize para separação de ambientes.

### Base (`apps/<app>/base/`)

Contém os manifests genéricos, sem namespace e sem configurações de ambiente:

- `deployment.yaml` — spec do container, portas, health checks
- `service.yaml` — ClusterIP service
- `kustomization.yaml` — lista os recursos do base

### Overlay (`apps/<app>/overlays/prod/`)

Contém as configurações específicas do ambiente de produção:

| Arquivo | Descrição | Presente em |
|---------|-----------|-------------|
| `kustomization.yaml` | Namespace, referência ao base, tag da imagem | Todos |
| `configmap.yaml` | Variáveis de ambiente (URLs, portas, configs) | Todos |
| `ingress.yaml` | Regras de roteamento HTTP | Todos (exceto loadtest) |
| `db-init-job.yaml` | Job de inicialização do banco de dados | auth, flags, targeting |
| `hpa.yaml` | Horizontal Pod Autoscaler | evaluation, analytics |
| `secrets.yaml` | Credenciais de banco de dados (gitignored) | auth, flags, targeting, evaluation, analytics |

### Imagens e tags

A tag da imagem é controlada pelo `kustomization.yaml` do overlay:

```yaml
images:
  - name: acrtogglemasterprod.azurecr.io/auth-service
    newTag: abc123def456     # SHA do commit (atualizado pela pipeline CI)
```

A pipeline CI atualiza a tag automaticamente via:

```bash
cd apps/auth/overlays/prod
kustomize edit set image acrtogglemasterprod.azurecr.io/auth-service:<COMMIT_SHA>
```

## Addons do Cluster

### Ingress NGINX

Ingress Controller para roteamento de tráfego externo para os serviços.

| Configuração | Valor |
|--------------|-------|
| Chart | `ingress-nginx` |
| Versão | `4.12.1` |
| Namespace | `ingress-nginx` |
| Tipo | Helm via ArgoCD multi-source |

### SonarQube

Servidor de análise de qualidade e cobertura de código, acessível via IP público para as pipelines CI.

| Configuração | Valor |
|--------------|-------|
| Chart | `sonarqube` |
| Versão | `2026.2.0` |
| Edição | Community |
| Namespace | `sonarqube` |
| Acesso | Ingress NGINX (sem hostname, acesso por IP) |
| PostgreSQL | `bitnami/postgresql:16.6.0-debian-12-r6` |
| Usuário admin | `admin` |

## ApplicationSet (descoberta automática de apps)

O `apps-appset.yaml` usa um **Git Directory Generator** que descobre microsserviços automaticamente:

```yaml
generators:
  - git:
      repoURL: https://github.com/freitasleoalves/fiap-tc-3-gitops.git
      revision: HEAD
      directories:
        - path: apps/*
```

Para adicionar um novo microsserviço, basta criar o diretório `apps/<novo-servico>/` com a estrutura `base/` e `overlays/prod/`. O ArgoCD criará a Application automaticamente.

### Convenção de nomes

| Variável | Exemplo | Descrição |
|----------|---------|-----------|
| `path.basename` | `auth` | Nome do diretório da app |
| Application name | `app-auth` | Prefixo `app-` + basename |
| Namespace | `togglemaster-auth` | Prefixo `togglemaster-` + basename |

## Sync Policy

Todas as Applications estão configuradas com:

```yaml
syncPolicy:
  automated:
    selfHeal: true    # Reverte mudanças manuais feitas no cluster
    prune: true       # Remove recursos que foram deletados do Git
  syncOptions:
    - CreateNamespace=true   # Cria o namespace se não existir
```

- **selfHeal** — Se alguém alterar um recurso manualmente no cluster, o ArgoCD reverte para o estado definido no Git
- **prune** — Se um manifesto for removido do Git, o recurso correspondente é removido do cluster

## Microsserviços

| Serviço | Linguagem | Imagem ACR | Namespace |
|---------|-----------|-----------|-----------|
| auth | Go 1.21 | `acrtogglemasterprod.azurecr.io/auth-service` | `togglemaster-auth` |
| flags | Python 3.11 | `acrtogglemasterprod.azurecr.io/flag-service` | `togglemaster-flags` |
| targeting | Python 3.11 | `acrtogglemasterprod.azurecr.io/targeting-service` | `togglemaster-targeting` |
| evaluation | Go 1.21 | `acrtogglemasterprod.azurecr.io/evaluation-service` | `togglemaster-evaluation` |
| analytics | Python 3.11 | `acrtogglemasterprod.azurecr.io/analytics-service` | `togglemaster-analytics` |
| loadtest | k6 | — | `togglemaster-loadtest` |

## Secrets

Os arquivos `secrets.yaml` estão no `.gitignore` e **não são versionados**. Para criá-los no cluster:

```bash
# Exemplo para o auth-service
kubectl create secret generic db-secrets \
  --namespace=togglemaster-auth \
  --from-literal=DB_HOST=<HOST> \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_USER=<USER> \
  --from-literal=DB_PASSWORD=<PASSWORD> \
  --from-literal=DB_NAME=auth_db
```

Repita para os namespaces `togglemaster-flags`, `togglemaster-targeting`, `togglemaster-evaluation` e `togglemaster-analytics` com as credenciais correspondentes.

## Comandos úteis

### ArgoCD CLI

```bash
# Login no ArgoCD
argocd login <ARGOCD_SERVER> --username admin --password <PASSWORD>

# Listar todas as applications
argocd app list

# Ver status de uma application
argocd app get app-auth

# Forçar sync de uma application
argocd app sync app-auth

# Ver diff entre Git e cluster
argocd app diff app-auth
```

### kubectl

```bash
# Ver pods de um serviço
kubectl get pods -n togglemaster-auth

# Ver logs de um pod
kubectl logs -n togglemaster-flags -l app=flag-service

# Ver status dos ingresses
kubectl get ingress -A

# Ver todos os namespaces do projeto
kubectl get ns | grep togglemaster
```

### Kustomize

```bash
# Visualizar os manifests renderizados de uma app
kustomize build apps/auth/overlays/prod

# Atualizar tag da imagem (feito pela pipeline CI)
cd apps/auth/overlays/prod
kustomize edit set image acrtogglemasterprod.azurecr.io/auth-service:novo-sha
```

## Adicionar novo microsserviço

1. Crie a estrutura de diretórios:

```bash
mkdir -p apps/novo-servico/base apps/novo-servico/overlays/prod
```

2. Crie os arquivos base (`deployment.yaml`, `service.yaml`, `kustomization.yaml`)

3. Crie os arquivos do overlay (`kustomization.yaml`, `configmap.yaml`, `ingress.yaml`)

4. Faça push — o ApplicationSet detecta o novo diretório e cria a Application automaticamente

## Fluxo CI/CD completo

```
┌──────────────┐     ┌──────────────────────────────────────────┐
│  Dev Push    │     │           GitHub Actions CI              │
│  (serviço)   │────►│  lint ──┐                                │
│              │     │         ├──► sonarqube                   │
│              │     │  test ──┤                                │
│              │     │         └──► build ──► trivy ──► push ACR│
└──────────────┘     └─────────────────────────┬────────────────┘
                                               │
                                               ▼
                     ┌──────────────────────────────────────────┐
                     │         update-gitops job                │
                     │  kustomize edit set image <sha>          │
                     │  git commit + push para este repo        │
                     └─────────────────────────┬────────────────┘
                                               │
                                               ▼
                     ┌──────────────────────────────────────────┐
                     │              ArgoCD                      │
                     │  Detecta mudança ──► Sync ──► AKS       │
                     └──────────────────────────────────────────┘
```
