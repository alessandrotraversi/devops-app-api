# 03 — Evoluindo: Empacotando os manifestos

**Onde estamos:** uma pasta com YAMLs soltos, aplicados com `kubectl apply -f k8s/`, valores fixos e um único ambiente.

---

## 🥉 Nível 1 — Ordem e organização

- **Um recurso por arquivo** (ou recursos relacionados claramente juntos) — evita "Services escondidos".
- **Garanta a ordem:** aplique o namespace primeiro, ou prefixe os arquivos (`00-namespace.yaml`, `10-config.yaml`, `20-deployment.yaml`...).
- **Valide antes de aplicar:**
  ```bash
  kubectl apply -f k8s/ --dry-run=server   # o próprio cluster valida
  kubeconform -summary k8s/                # valida contra o schema
  ```
- **Labels recomendadas** em tudo: `app.kubernetes.io/name`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`.

---

## 🥈 Nível 2 — Kustomize (base + ambientes)

O Kustomize já vem dentro do `kubectl`. Ideia: uma **base** comum e **overlays** (camadas) que só mudam o que é diferente em cada ambiente.

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml     # 1 réplica, tag "dev-abc123"
    └── prod/
        └── kustomization.yaml     # 3 réplicas, mais recursos
```

```yaml
# overlays/dev/kustomization.yaml
resources: [../../base]
namespace: minha-api-dev
images:
  - name: minha-api
    newTag: abc1234          # o pipeline troca isso
replicas:
  - name: minha-api
    count: 1
```

```bash
kubectl apply -k k8s/overlays/dev
```

O Kustomize cuida da ordem (namespace primeiro) e permite trocar a tag de imagem sem editar o Deployment.

---

## 🥇 Nível 3 — Helm

O **Helm** transforma os manifestos em um "pacote instalável" (chart) com templates e um arquivo de valores. Vale quando:

- você vai **distribuir** a aplicação para outras pessoas/times;
- precisa de lógica (condicionais, laços);
- quer versionar e fazer rollback do **conjunto** (`helm rollback`).

Para uma única aplicação interna, Kustomize costuma bastar. Use Helm para **instalar dependências** (Postgres operator, Prometheus, Argo CD).

---

## Como validar

```bash
kubectl kustomize k8s/overlays/dev | less     # veja o YAML final gerado
kubectl diff -k k8s/overlays/dev              # o que mudaria no cluster
```
