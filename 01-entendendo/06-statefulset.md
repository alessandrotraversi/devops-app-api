# 06 — StatefulSet (Postgres e Redis)

## O que é?

O Deployment trata pods como **gado**: todos iguais, intercambiáveis. Isso é ótimo para APIs, mas péssimo para bancos de dados, que precisam:

1. **Nome estável** (`banco-0`, `banco-1`... sempre os mesmos).
2. **Disco próprio** que sobrevive à morte do pod.
3. **Ordem** para subir e descer (primeiro o 0, depois o 1...).

O **StatefulSet** dá exatamente isso.

> **Analogia:** Deployment = garçons de um buffet (qualquer um serve). StatefulSet = cofres de um banco, cada um com número e conteúdo próprios.

## Os ingredientes de armazenamento

| Peça | O que é |
|---|---|
| **PersistentVolume (PV)** | Um disco real (local, EBS, etc.) |
| **PersistentVolumeClaim (PVC)** | Um "pedido" de disco: "quero 1 GiB" |
| **StorageClass** | O "fornecedor" que cria o disco automaticamente quando há um pedido |
| **volumeClaimTemplates** | Dentro do StatefulSet: "para cada pod, crie um PVC assim" |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: meu-banco
spec:
  serviceName: meu-banco      # Service "headless" que dá DNS a cada pod
  replicas: 1
  selector: { matchLabels: { app: meu-banco } }
  template:
    metadata: { labels: { app: meu-banco } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: dados
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata: { name: dados }
      spec:
        accessModes: [ReadWriteOnce]   # um nó por vez pode montar
        resources: { requests: { storage: 1Gi } }
```

Se o pod `meu-banco-0` morrer, o novo `meu-banco-0` **reconecta no mesmo disco**. Os dados ficam.

## Como aparece no app-api

### Postgres
- 1 réplica, imagem Alpine (menor).
- Usuário e senha vêm do Secret; o nome do banco está escrito direto.
- Disco de 1 GiB usando a StorageClass `local-path` (a padrão do k3s, que grava numa pasta do nó).
- Probes usam o comando `pg_isready` (o próprio Postgres responde se aceita conexões). Repare: aqui as probes são do tipo **`exec`** (roda um comando dentro do container), diferente do **`httpGet`** da API.

### Redis
- 1 réplica, sem disco (sem `volumeClaimTemplates`). Ou seja: se o pod reiniciar, **o cache é perdido** — aceitável para cache, ruim se for usado como fila ou sessão.
- Probes com `redis-cli ping`.
- Sem senha.

## O Service "headless"

O campo `serviceName` de um StatefulSet deveria apontar para um Service com `clusterIP: None`. Isso faz o DNS criar um nome **por pod** (`meu-banco-0.meu-banco`), essencial quando há várias réplicas (ex.: "escreva só no primário").

⚠️ No app-api, o `serviceName` aponta para um Service comum (com IP). Com 1 réplica funciona, mas não é o padrão. Ver [capítulo 14](14-o-que-a-ia-deixou-passar.md).

## Banco dentro do Kubernetes: vale a pena?

Para estudo e ambiente local: **sim**, é ótimo para aprender.
Para produção: um StatefulSet "cru" não faz **backup**, **failover** nem **atualização de versão** sozinho. Ver [02-evoluindo/06](../02-evoluindo/06-banco-de-dados.md).

## Comandos para explorar

```bash
kubectl get sts,pvc,pv -n meu-projeto
kubectl exec -it meu-banco-0 -- psql -U <usuario> -d <banco>
kubectl delete pod meu-banco-0       # veja ele voltar com o mesmo nome e os mesmos dados
```

Próximo: [07 — HPA](07-hpa.md)
