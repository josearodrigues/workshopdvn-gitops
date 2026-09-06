# workshopdvn‑gitops

> **Repositório de artefatos GitOps** – sobreposições Kustomize, manifests do Argo CD e descritores de implantação Kubernetes que alimentam a pilha da aplicação *Workshop DVN*.

---

## 📚 Visão geral

`workshopdvn‑gitops` contém os ativos **infra‑as‑code** que descrevem como os componentes **backend** e **frontend** da solução *Workshop DVN* são provisionados em um cluster Kubernetes.  A pasta segue o padrão **GitOps**:

1. **Fonte da verdade** – Todos os manifests residem no Git.
2. **Reconciliação automática** – Argo CD observa este repositório e aplica os manifests sempre que eles mudam.
3. **Configuração declarativa** – Kustomize é usado para aplicar ajustes específicos de ambiente (tags de imagens, contagem de réplicas, etc.) sobre os manifests base.

O projeto também inclui outras pastas iniciadas por `workshop*` (por exemplo, `workshopdvn‑app*`, `workshopdvn‑iac`). Essas contêm o código‑fonte da aplicação e definições de IaC; esta pasta é responsável exclusivamente pela **entrega contínua**.

---

## 📂 Estrutura de diretórios

```text
workshopdvn-gitops/
├─ .git/                 # Metadados do Git (gerado automaticamente)
├─ argocd/               # Manifests opcionais de Application do Argo CD (caso prefira bootstrapping via CR)
├─ backend/
│   ├─ deploy.yml        # Deployment Kubernetes para o serviço backend
│   └─ service.yml       # Service que expõe os pods do backend
├─ frontend/
│   ├─ deploy.yml        # Deployment Kubernetes para o frontend (SPA)
│   └─ service.yml       # Service que expõe os pods do frontend
├─ kustomization.yml     # Arquivo Kustomize raiz – agrupa recursos e sobrescreve imagens
└─ README.md            # ← este documento
```

* **`backend/`** – Manifests que criam um Deployment rodando a imagem Docker `.../backend` e um Service ClusterIP.
* **`frontend/`** – Mesmo padrão para o componente UI.
* **`kustomization.yml`** – Coleta os quatro arquivos de recurso acima e substitui as tags de imagem pelos SHA exatos construídos pelo pipeline CI.
* **`argocd/`** – (Placeholder) pode armazenar um `Application` CR caso deseje bootstrapping do Argo CD via Helm chart ou outro repositório Git.

---

## ⚙️ Pré‑requisitos

| Ferramenta | Versão mínima |
|------------|---------------|
| **kubectl** | 1.27 |
| **kustomize** (integrado ao `kubectl` v1.27+) | – |
| **Argo CD** | 2.7 |
| **Docker** (para construir imagens) | 24.0 |
| **Git** | 2.40 |

É necessário ter acesso ao registro **ECR** referenciado na seção `images:` do `kustomization.yml` (credenciais AWS com `ecr:GetAuthorizationToken`).

---

## 🚀 Deploy com Kustomize (manual)

```bash
# Clone o repositório (caso ainda não tenha)
git clone https://github.com/josearodrigues/workshopdvn-gitops.git
cd workshopdvn-gitops

# Renderiza os manifests finais (substitui as tags de imagem pelos valores definidos em kustomization.yml)
kubectl kustomize . > rendered-manifests.yml

# Aplica no cluster de destino
kubectl apply -f rendered-manifests.yml
```

> **Dica** – Você pode encadear a renderização diretamente ao `apply` usando `kubectl apply -k .`:
>
> ```bash
> kubectl apply -k .
> ```

---

## 📈 Entrega contínua com Argo CD

1. **Crie uma Application do Argo CD** (operação única):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: workshopdvn
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/josearodrigues/workshopdvn-gitops.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: workshopdvn
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

   Aplique o manifest com `kubectl apply -f application.yml` (ou coloque o arquivo em `argocd/` e deixe o Argo CD carregá‑lo).

2. O Argo CD monitorará continuamente o repositório Git. Sempre que um novo commit atualizar as tags de imagem (geralmente o pipeline CI altera o SHA), o Argo CD sincroniza as alterações no cluster de forma automática.

---

## 🔄 Atualizando imagens de containers

O pipeline CI (GitLab, GitHub Actions, etc.) deve:

1. Construir as imagens Docker do **backend** e do **frontend**.
2. Publicá‑las no repositório ECR indicado em `kustomization.yml`.
3. Commitar um novo `kustomization.yml` (ou um overlay separado) com os valores atualizados de `newTag:` (o SHA da imagem).
4. Push do commit – Argo CD detecta a mudança e executa um rollout.

Caso precise sobrescrever rapidamente manualmente:

```bash
kubectl set image deployment/backend backend=381491977261.dkr.ecr.us-east-1.amazonaws.com/workshop-na-nuvem/production/backend:<novo‑sha> -n workshopdvn
kubectl set image deployment/frontend frontend=381491977261.dkr.ecr.us-east-1.amazonaws.com/workshop-na-nuvem/production/frontend:<novo‑sha> -n workshopdvn
```

---

## 🛠️ Contribuindo

1. Crie um *branch* a partir de `main` com um nome descritivo, por exemplo `feature/add‑health‑checks`.
2. Faça as alterações (novo overlay, ajuste de limites, etc.).
3. Execute `kubectl kustomize .` localmente para validar os manifests renderizados.
4. Abra um **Pull Request** contra `main`. O CI fará lint dos arquivos YAML com `yamllint` e validação contra o esquema Kubernetes via `kubeval`.
5. Após aprovação e merge, o Argo CD aplicará automaticamente as mudanças.

---

## 📄 Licença

Esta configuração GitOps está licenciada sob a **MIT License** – sinta‑se livre para fork‑ar, adaptar e reutilizar em seus próprios projetos.