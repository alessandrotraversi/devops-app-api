# 06 — Evoluindo: Banco de dados

**Onde estamos:** Postgres num StatefulSet com 1 réplica e disco local, sem backup. Redis sem disco e sem senha.

---

## 🥉 Nível 1 — Acertar o básico

- **Service headless** para o StatefulSet (`clusterIP: None`) + um Service normal para a aplicação, se quiser.
- **Probe sem nome fixo:** use a variável de ambiente que já existe no container:
  ```yaml
  readinessProbe:
    exec:
      command: ["sh", "-c", "pg_isready -U \"$POSTGRES_USER\" -d \"$POSTGRES_DB\""]
  ```
- **`PGDATA` em subpasta** (`/var/lib/postgresql/data/pgdata`): evita erro quando o volume já vem com uma pasta `lost+found`.
- Alinhar a versão (README × manifesto).
- **Redis:** decidir — é só cache? Então tudo bem perder dados, mas documente. Usa como fila/sessão? Então precisa de persistência (volume + AOF).

---

## 🥈 Nível 2 — Backup e restauração

> Backup que nunca foi restaurado não é backup.

Opção simples: um **CronJob** que roda `pg_dump` e salva num volume ou bucket.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: backup-banco }
spec:
  schedule: "0 3 * * *"        # todo dia às 3h
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: dump
              image: postgres:16
              command: ["sh", "-c", "pg_dump -h meu-banco -U \"$PGUSER\" app > /backup/$(date +%F).sql"]
              # env (PGUSER/PGPASSWORD) via Secret, volume /backup via PVC
```

E teste a restauração num namespace separado de tempos em tempos.

---

## 🥇 Nível 3 — Operator ou banco gerenciado

Rodar Postgres "na mão" não dá: failover, réplicas de leitura, backup contínuo (PITR), atualização de versão.

| Caminho | Quando escolher |
|---|---|
| **Operator** (CloudNativePG, Zalando) | Quer ficar dentro do Kubernetes; o operator cuida de réplicas, failover e backup |
| **Banco gerenciado** (RDS, Cloud SQL, Azure DB) | Em nuvem, quase sempre a melhor escolha: o problema vira do provedor |

Exemplo do nível de abstração com CloudNativePG:
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: meu-banco }
spec:
  instances: 3          # 1 primário + 2 réplicas, failover automático
  storage: { size: 10Gi }
```

Migrations (Alembic) também evoluem: rodar como **Job** antes do deploy, em vez de na inicialização da aplicação.

---

## Como validar

```bash
kubectl delete pod meu-banco-0     # dados continuam?
# Restaure um backup em outro namespace e conte as linhas
```
