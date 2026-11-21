# 07 — Evoluindo: Resiliência

**Onde estamos:** 2 réplicas, HPA e probes, mas a readiness não olha as dependências e não há proteção contra manutenções do cluster.

---

## 🥉 Nível 1 — Probes com papéis certos

| Probe | Endpoint | Deve checar |
|---|---|---|
| liveness | `/health` | Só o próprio processo. **Nunca** o banco (senão uma queda do banco reinicia todos os pods à toa) |
| readiness | `/ready` | Dependências essenciais (banco). Falhou → sai do tráfego |
| startup | `/health` | Dá tempo para a aplicação ligar (migrations, cache) sem a liveness matar antes |

```yaml
startupProbe:
  httpGet: { path: /health, port: 8000 }
  periodSeconds: 5
  failureThreshold: 30       # até 150 s para ligar
```

Remova a métrica inválida do HPA e mantenha só a de CPU.

---

## 🥈 Nível 2 — Atualizações e manutenções sem susto

**Estratégia de rollout explícita:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0     # nunca fica com menos pods que o desejado
    maxSurge: 1           # sobe 1 a mais por vez
```

**PodDisruptionBudget** — protege contra *manutenções* (ex.: drenar um nó):
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: minha-api }
spec:
  minAvailable: 1
  selector: { matchLabels: { app: minha-api } }
```

**Desligamento gracioso:** o pod recebe um sinal de "vou te desligar"; a aplicação deve terminar as requisições em andamento. Ajuste `terminationGracePeriodSeconds` e, se preciso, um `preStop` com uma pequena pausa para o Service tirar o pod antes de ele parar.

**Recursos medidos, não chutados:** observe `kubectl top` sob carga e ajuste `requests` para perto do uso real.

---

## 🥇 Nível 3 — Falhas de infraestrutura

- **Espalhar réplicas** entre nós e zonas:
  ```yaml
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector: { matchLabels: { app: minha-api } }
  ```
- **Escalar por métricas de negócio** (requisições/s, tamanho de fila) com KEDA ou Prometheus Adapter.
- **Caos controlado** (Chaos Mesh, LitmusChaos): matar pods e derrubar o banco de propósito para ver se o sistema se comporta como o runbook diz.

---

## Como validar

```bash
# Derrube o banco e veja os pods da API ficarem "0/1 Ready" sem reiniciar
kubectl scale sts meu-banco --replicas=0
kubectl get pods -l app=minha-api -w
```
