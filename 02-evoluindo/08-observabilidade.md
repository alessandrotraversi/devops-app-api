# 08 — Evoluindo: Observabilidade

**Onde estamos:** a aplicação manda telemetria via OTel para um coletor; existem alertas, mas os de erro/latência não têm dados, e os limites não batem com os SLOs.

---

## 🥉 Nível 1 — Fazer os dados existirem

Escolha **um** caminho para as métricas HTTP chegarem ao Prometheus:

| Caminho | Como |
|---|---|
| **A. Via coletor OTel** | A aplicação envia por OTLP → o coletor expõe no formato Prometheus → o Prometheus raspa o **coletor** |
| **B. Direto da aplicação** | A aplicação expõe `/metrics` no formato Prometheus (ex.: `prometheus-fastapi-instrumentator`) → o Prometheus raspa a **aplicação** |

Em ambos, diga ao Prometheus onde buscar com um **ServiceMonitor**:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: { name: minha-api }
spec:
  selector: { matchLabels: { app: minha-api } }
  endpoints:
    - port: http            # o Service precisa ter a porta com esse nome
      path: /metrics
```

Depois: abra o Prometheus, **descubra o nome real** da métrica e use esse mesmo nome no runbook e nos alertas. Renomeie o `/metrics` atual (JSON) para algo como `/status`.

Adicione o processador `batch` no coletor e um destino de verdade para traces (Jaeger ou Tempo).

---

## 🥈 Nível 2 — Alertas derivados dos SLOs

Um só lugar define o SLO, e os alertas saem dele. Em vez de "erro > X%", alerte sobre **velocidade de consumo do error budget** (*burn rate*), com **duas janelas** para evitar falso positivo:

| Severidade | Burn rate | Janela longa | Janela curta | Significa |
|---|---|---|---|---|
| critical (acordar alguém) | 14,4× | 1 h | 5 min | 2% do budget do mês em 1 h |
| warning (ticket) | 6× | 6 h | 30 min | 5% do budget em 6 h |

Ferramentas que **geram** essas regras a partir de um SLO declarado: **Sloth**, **Pyrra**, **OpenSLO**.

Também:
- Todo alerta deve ter uma annotation `runbook_url` apontando para a seção certa do runbook.
- Configure o **Alertmanager** para rotear por `severity` (e usar `blast_radius` na priorização).

---

## 🥇 Nível 3 — Correlacionar os três sinais

- **Logs estruturados (JSON)** com `trace_id` em cada linha → do alerta ao trace ao log em 3 cliques.
- **Exemplars**: um ponto da métrica de latência linka para um trace real lento.
- Stack integrada (ex.: Grafana + Prometheus/Mimir + Loki + Tempo).
- **Atributos de ambiente** (`deployment.environment`, versão) vindos do ConfigMap/pipeline, não fixos na imagem.
- Dashboards como código (versionados no Git).

---

## Como validar

```bash
# Gere erros de propósito e confira se o alerta passa por pending → firing
for i in $(seq 1 500); do curl -s http://api.exemplo.test/rota-que-da-500 >/dev/null; done
```
