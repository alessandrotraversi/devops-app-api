# 09 — PrometheusRule (alertas)

## O que é?

O **Prometheus** coleta métricas (números ao longo do tempo) e avalia regras. Uma **PrometheusRule** é um recurso do Kubernetes (não nativo — vem do *Prometheus Operator*) onde você escreve:

> "Se **esta consulta** for verdadeira por **X minutos**, dispare **este alerta**."

> **Analogia:** o alarme de fumaça. Não dispara com uma faísca (o `for:` evita alarmes falsos), mas dispara se a fumaça persistir.

```yaml
apiVersion: monitoring.coreos.com/v1   # repare: não é "v1" puro → é um CRD
kind: PrometheusRule
metadata:
  name: alertas-da-api
spec:
  groups:
    - name: api
      rules:
        - alert: ApiComMuitosErros
          expr: <consulta PromQL> > 0.05
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Taxa de erro acima de 5%"
```

## Anatomia de uma regra

| Campo | Para que serve |
|---|---|
| `alert` | Nome do alerta |
| `expr` | Consulta em **PromQL**; se retornar algo, a condição é verdadeira |
| `for` | Quanto tempo a condição precisa durar antes de disparar (`0m` = imediato) |
| `labels` | Etiquetas usadas para **roteamento** (ex.: `critical` → acorda alguém; `warning` → só Slack) |
| `annotations` | Texto para humanos; aceita variáveis como `{{ $value }}` |

## Os 6 alertas do app-api, por conceito

| Conceito | Ideia | Fonte da métrica |
|---|---|---|
| **Taxa de erro** | Respostas 5xx ÷ total de respostas > limite | Métricas da aplicação |
| **Latência P95** | 95% das requisições abaixo de N segundos? | Histograma da aplicação |
| **Reinícios** | Muitos reinícios do container em 15 min | `kube-state-metrics` |
| **Pod não pronto** | Readiness falhando por alguns minutos | `kube-state-metrics` |
| **OOMKilled** | Container morto por estourar memória | `kube-state-metrics` |
| **CrashLoopBackOff** | Container que nasce e morre em loop | `kube-state-metrics` |

Além de `severity`, o app-api usa uma label `blast_radius` (raio de impacto: alto/médio) — boa prática para priorizar.

## Mini aula de PromQL

- `rate(contador[5m])` → velocidade média de crescimento de um contador nos últimos 5 min (ex.: requisições por segundo).
- `sum(...)` → soma tudo (de todos os pods).
- `increase(contador[15m])` → quanto aumentou em 15 min.
- `histogram_quantile(0.95, ...)` → o **percentil 95** a partir de um histograma.

**Por que P95 e não média?** A média esconde os lentos. Se 95 requisições levam 10 ms e 5 levam 10 s, a média é ~0,5 s (parece ok), mas 5% dos usuários estão sofrendo. O P95 mostra isso.

## Duas famílias de alerta

- **Sintoma** (o que o usuário sente): erro e latência. São os mais importantes.
- **Causa** (o que está acontecendo por dentro): reinício, OOM, CrashLoop. Ajudam no diagnóstico, mas podem ser barulhentos.

## Pontos de atenção

⚠️ Para os alertas de erro e latência funcionarem, a aplicação precisa **expor** métricas no formato do Prometheus e o Prometheus precisa **saber onde buscá-las** (um `ServiceMonitor`). Nenhuma das duas coisas existe hoje no app-api, e os limites dos alertas não batem com os SLOs do runbook. Ver [capítulo 14](14-o-que-a-ia-deixou-passar.md).

## Comandos para explorar

```bash
kubectl get prometheusrules -A
# Na UI do Prometheus (porta 9090): aba "Alerts" mostra estado inactive/pending/firing
```

- **inactive**: condição falsa.
- **pending**: condição verdadeira, mas ainda dentro do `for`.
- **firing**: disparado.

Próximo: [10 — Dockerfile](10-dockerfile.md)
