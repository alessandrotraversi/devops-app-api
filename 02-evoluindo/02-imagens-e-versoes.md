# 02 — Evoluindo: Imagens e versões

**Onde estamos:** o Deployment usa a tag `latest` com `IfNotPresent`, então um deploy pode não trocar a versão que está rodando.

---

## 🥉 Nível 1 — Uma tag por versão

**Regra de ouro: tag de imagem é imutável.** Uma tag deve sempre apontar para o mesmo conteúdo.

No pipeline, use o hash do commit e atualize o Deployment com ele:
```bash
TAG=$(git rev-parse --short HEAD)
docker build -t minha-api:$TAG .
k3d image import minha-api:$TAG --cluster meu-cluster
kubectl set image deployment/minha-api minha-api=minha-api:$TAG
```
Agora o YAML **muda** a cada deploy → o Kubernetes faz rollout → `kubectl rollout undo` passa a fazer sentido, porque cada revisão aponta para uma imagem diferente.

Bônus rápido:
- Crie um `.dockerignore` (`.git`, `tests/`, `.venv`, `__pycache__`, `*.md`).
- Remova o `curl` do estágio de build, e da imagem final se não usar o HEALTHCHECK do Docker.
- Tire da imagem as variáveis que dependem do ambiente (vão para o ConfigMap).

---

## 🥈 Nível 2 — Registry

Em vez de copiar a imagem para dentro do k3d, publique num **registry** (GitHub Container Registry, Docker Hub, ou um registry local criado pelo próprio k3d):

```bash
k3d registry create meu-registry --port 5050
k3d cluster create estudo --registry-use k3d-meu-registry:5050
```

Isso aproxima o fluxo local do fluxo real de nuvem, onde o cluster **baixa** a imagem.

Adote também **versão semântica** para releases (`1.4.2`) além do hash de commit.

---

## 🥇 Nível 3 — Cadeia de suprimentos

- **Fixar por digest** (`minha-api@sha256:...`): garante byte a byte qual imagem roda.
- **Fixar a imagem base** por versão completa ou digest, e atualizar com Renovate/Dependabot.
- **Assinar** a imagem (cosign) e exigir assinatura no cluster (Kyverno/policy-controller).
- Gerar **SBOM** (lista de tudo que está dentro da imagem) com syft/trivy.
- Considerar imagens base mínimas (*distroless*, Chainguard).

---

## Como validar

```bash
kubectl get deploy minha-api -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl rollout history deploy/minha-api   # deve crescer a cada deploy
```
