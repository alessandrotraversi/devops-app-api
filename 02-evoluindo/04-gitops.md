# 04 — Evoluindo: GitOps

**Onde estamos:** o pipeline **empurra** mudanças para o cluster com `kubectl apply` (modelo *push*). Se alguém alterar algo à mão no cluster, ninguém fica sabendo.

---

## O conceito

**GitOps** = o Git é a **fonte da verdade**. Um agente **dentro** do cluster fica olhando o repositório e aplica o que estiver lá (modelo *pull*). Se o cluster divergir do Git, o agente avisa ou corrige.

> **Analogia:** em vez de um entregador que você chama (push), é um assinante que confere o jornal todo dia e atualiza a casa conforme o que leu (pull).

```
Push (hoje):   Dev → Git → Pipeline ──kubectl──► Cluster
Pull (GitOps): Dev → Git ◄──observa── Argo CD (dentro do cluster) ──aplica──► Cluster
```

Vantagens:
- O pipeline **não precisa** de credencial do cluster.
- Todo estado tem histórico (é commit).
- Rollback = `git revert`.
- Detecção de *drift* (mudanças manuais).

---

## 🥈 Nível 2 — Argo CD local

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Uma `Application` diz ao Argo CD o que observar:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minha-api-dev
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/<voce>/<repo>
    path: k8s/overlays/dev
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: minha-api-dev
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true]
```

O pipeline passa a fazer só: **build → push da imagem → commit atualizando a tag no overlay**. O Argo CD faz o resto.

Pré-requisito: [Kustomize](03-empacotando-manifestos.md) e [registry](02-imagens-e-versoes.md).

---

## 🥇 Nível 3 — Promoção entre ambientes

- Um overlay por ambiente (`dev`, `staging`, `prod`).
- Promover = abrir PR copiando a tag de `dev` para `staging` (auditável, com aprovação).
- Ferramentas como **Argo CD Image Updater** ou **Kargo** automatizam essa promoção.
- Alternativa ao Argo CD: **Flux**.

---

## Como validar

```bash
kubectl scale deploy/minha-api --replicas=5   # mudança manual
# Com selfHeal ligado, em segundos o Argo CD volta para o valor do Git.
```
