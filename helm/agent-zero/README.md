# Agent Zero Helm Chart

Este chart instala o [Agent Zero](https://agent-zero.ai) em um cluster Kubernetes.

## Pré-requisitos

- Kubernetes >= 1.21
- Helm 3.x
- PV provisioner (se `persistence.enabled=true`)

## Validação local (Kind)

Para validar o chart antes de publicar, use um cluster [Kind](https://kind.sigs.k8s.io/) local:

```bash
# Criar cluster
kind create cluster --name agent-zero-validate

# Empacotar e instalar (a partir da raiz do repositório)
helm package helm/agent-zero -d /tmp/helm-agent-zero
helm install agent-zero /tmp/helm-agent-zero/agent-zero-0.1.0.tgz --create-namespace --namespace agent-zero-demo

# Verificar recursos
kubectl -n agent-zero-demo get all,pvc
```

**Nota:** A imagem oficial usa setuid para o processo SearXNG. O chart define `allowPrivilegeEscalation: true` e adiciona apenas as capabilities SETUID, SETGID e NET_RAW. Em clusters com PSA muito restrito pode ser necessário relaxar políticas.

## Instalação

### Repositório Helm (após publicação)

```bash
helm repo add agent-zero https://agent0ai.github.io/agent-zero
helm repo update
helm install agent-zero agent-zero/agent-zero
```

### A partir do repositório local

```bash
helm install agent-zero ./helm/agent-zero
```

### Com valores customizados

```bash
helm install agent-zero ./helm/agent-zero -f my-values.yaml
```

## Configuração

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `image.repository` | Imagem Docker | `agent0ai/agent-zero` |
| `image.tag` | Tag da imagem | `latest` |
| `replicaCount` | Número de réplicas (recomendado 1) | `1` |
| `service.type` | Tipo do Service | `ClusterIP` |
| `service.port` | Porta do Service | `80` |
| `persistence.enabled` | Habilitar PVC para `usr/` (settings, workdir, plugins) | `true` |
| `persistence.size` | Tamanho do PVC | `10Gi` |
| `persistence.storageClass` | StorageClass do PVC | `""` |
| `persistence.existingClaim` | Nome de um PVC existente | `""` |
| `env` | Variáveis de ambiente (ver abaixo) | `{}` |
| `existingSecret` | Secret existente para senhas/API keys | `""` |
| `ingress.enabled` | Habilitar Ingress | `false` |
| `resources` | Requests/limits do container | Ver `values.yaml` |

### Segurança e capabilities

O chart faz **drop: [ALL]** e adiciona só o necessário:

| Capability | Uso |
|------------|-----|
| **SETUID** / **SETGID** | Supervisord rodar run_searxng como UID 995 (setuid/setgroups). |
| **NET_RAW** | Raw sockets para ferramentas de scan de rede (ex.: nmap). |

**Risco de não dropar capabilities:** sem `drop: [ALL]`, o container fica com o conjunto padrão do runtime; dropar tudo e adicionar só o necessário reduz a superfície de ataque.

**NET_RAW:** permite criar raw sockets (ex.: nmap com SYN scan, ping). Use apenas se o agente precisar de varredura de rede; em ambientes restritos pode ser removido com `--set securityContext.capabilities.add='{SETUID,SETGID}'` (ou override em values).

### Variáveis de ambiente importantes

Configure via `env` em `values.yaml` ou use um Secret em `existingSecret`:

- **`AUTH_LOGIN`** / **`AUTH_PASSWORD`** – Login da UI (Basic Auth).
- **`FLASK_SECRET_KEY`** – Chave de sessão do Flask.
- **`ALLOWED_ORIGINS`** – Origens permitidas (CSRF), separadas por vírgula.
- **`A0_SET_<name>`** – Override de qualquer setting (ex.: `A0_SET_chat_model_provider: "openai"`).

Exemplo:

```yaml
env:
  AUTH_LOGIN: "admin"
  AUTH_PASSWORD: "changeme"
  ALLOWED_ORIGINS: "https://agent.example.com"
  A0_SET_chat_model_provider: "openai"
  A0_SET_chat_model_name: "gpt-4o"
```

Para chaves de API e dados sensíveis, use um Secret e defina `existingSecret` com o nome desse Secret.

## Acesso após instalação

- **ClusterIP (padrão):** `kubectl port-forward svc/<release-name> 8080:80` e acesse http://localhost:8080.
- **Ingress:** configure `ingress.enabled=true` e os hosts em `ingress.hosts`.
- **LoadBalancer:** defina `service.type: LoadBalancer`.

## Persistência

Com `persistence.enabled=true`, o diretório `/a0/usr` (settings, workdir, uploads, plugins) é armazenado em um PVC. Para usar um PVC já existente, defina `persistence.existingClaim` com o nome do claim.

## Publicação do chart

O chart é empacotado e publicado automaticamente via GitHub Actions quando há alterações em `helm/agent-zero/`. O repositório Helm fica em:

- **URL do repositório:** https://agent0ai.github.io/agent-zero

Documentação de instalação e atualização: [docs/setup/helm.md](https://github.com/agent0ai/agent-zero/blob/main/docs/setup/helm.md).
