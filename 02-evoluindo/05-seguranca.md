# 05 — Evoluindo: Segurança

**Onde estamos:** a API já roda como não-root e com disco somente-leitura (ótimo começo). A NetworkPolicy é permissiva, banco e cache não têm proteção, e não há TLS.

---

## 🥉 Nível 1 — securityContext completo (em todos os pods)

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile: { type: RuntimeDefault }   # filtra chamadas de sistema perigosas
  containers:
    - name: minha-api
      securityContext:
        allowPrivilegeEscalation: false        # impede virar root via sudo/setuid
        readOnlyRootFilesystem: true
        capabilities: { drop: [ALL] }          # remove "superpoderes" do Linux
```

Aplique também no Postgres e no Redis (as imagens oficiais aceitam rodar como não-root com o usuário certo).

E: **`automountServiceAccountToken: false`** se o pod não precisa falar com a API do Kubernetes (a API não precisa). Isso evita que um invasor use o token do pod.

---

## 🥈 Nível 2 — Rede fechada por padrão

Passo 1 — **negar tudo** no namespace:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: negar-tudo }
spec:
  podSelector: {}               # todos os pods do namespace
  policyTypes: [Ingress, Egress]
```

Passo 2 — **liberar só o necessário**, uma regra por necessidade:

| Quem | Pode falar com | Porta |
|---|---|---|
| Ingress Controller (kube-system) | API | 8000 |
| API | Postgres | 5432 |
| API | Redis | 6379 |
| API | Coletor OTel (namespace de observabilidade) | 4317/4318 |
| Todos | DNS (kube-system) | 53 UDP/TCP |
| Prometheus | API | porta de métricas |

Use `namespaceSelector` com a label automática `kubernetes.io/metadata.name: <namespace>` em vez de `{}`.

Coloque **senha no Redis** (`requirepass`, vinda de um Secret).

---

## 🥇 Nível 3 — Postura de cluster

- **Pod Security Admission:** label no namespace que **recusa** pods inseguros:
  ```yaml
  metadata:
    labels:
      pod-security.kubernetes.io/enforce: restricted
  ```
- **TLS no Ingress** com **cert-manager** (Let's Encrypt em nuvem; CA local em dev).
- **RBAC** mínimo para o runner do pipeline (só pode mexer no seu namespace).
- **Políticas como código** (Kyverno/OPA Gatekeeper): "proibido `latest`", "obrigatório `resources`".
- **Runner self-hosted:** não rode PRs de forks nele.

---

## Como validar

```bash
trivy config k8s/
kube-linter lint k8s/
# Teste de rede: de um pod qualquer, o Redis deve estar inacessível
kubectl run teste --rm -it --image=busybox -n outro-ns -- nc -zv -w 3 redis.minha-api 6379
```
