# 13 — Observabilidade com OpenTelemetry

## O que é?

**Observabilidade** = conseguir responder "o que está acontecendo lá dentro?" sem precisar adicionar código novo. Ela se apoia em três sinais:

| Sinal | Pergunta | Exemplo |
|---|---|---|
| **Métricas** | "Quanto?" | requisições/s, % de erro, latência P95 |
| **Traces** | "Por onde passou e onde demorou?" | uma requisição: API 20 ms → banco 180 ms → Redis 2 ms |
| **Logs** | "O que exatamente aconteceu?" | `ERRO: timeout conectando no banco` |

## OpenTelemetry (OTel)

Um **padrão aberto** para gerar e enviar esses sinais. A vantagem: você instrumenta uma vez e pode mandar para qualquer ferramenta (Prometheus, Jaeger, Grafana, Datadog...).

O caminho dos dados:

```
 App (SDK OTel) ──OTLP──► Collector ──► destinos (Prometheus, Jaeger, console...)
```

- **OTLP**: o "idioma" do OTel. Porta **4317** = gRPC, **4318** = HTTP.
- **Collector**: um intermediário que recebe, processa e reenvia. Desacopla a aplicação das ferramentas.

## O arquivo do Collector, por conceito

Todo config de Collector tem 3 partes (+ processadores, opcionais):

| Parte | Papel | No app-api |
|---|---|---|
| `receivers` | Por onde os dados **entram** | OTLP em gRPC e HTTP |
| `exporters` | Para onde os dados **saem** | Endpoint para o Prometheus "raspar" + `debug` (imprime no console) |
| `service.pipelines` | Liga entradas a saídas, por sinal | traces → só console; métricas → Prometheus + console |

> **Analogia:** uma central dos Correios. Receivers são as caixas de coleta, exporters são os caminhões de entrega, pipelines são as rotas.

Observações:
- Traces vão só para o console — não há onde visualizá-los (Jaeger/Tempo). É um bom primeiro passo, mas não um destino final.
- Não há `processors` (ex.: `batch`, que agrupa envios e é recomendado sempre).

## O runbook

Um **runbook** é o manual de "o que fazer quando der problema": sintomas, comandos de diagnóstico, consultas e ações. O do app-api também define os **SLOs**.

### SLI, SLO, SLA e error budget

| Termo | O que é | Exemplo |
|---|---|---|
| **SLI** (indicador) | Uma medida | % de requisições com sucesso |
| **SLO** (objetivo) | A meta interna para o SLI | 99,9% de sucesso no mês |
| **SLA** (acordo) | Promessa contratual, com multa | 99,5% ou devolvemos dinheiro |
| **Error budget** | Quanto você **pode** falhar | 0,1% de um mês ≈ 43 minutos |

### Burn rate

"A que velocidade estou gastando meu orçamento de erro?" Burn rate 1 = gasto exatamente no ritmo para acabar no fim do mês. Burn rate 14,4 = acabaria em ~2 dias → alerta urgente. O runbook descreve esse conceito, mas os alertas do cap. 09 ainda não o implementam.

## Pontos de atenção

⚠️ Existem três "verdades" diferentes que deveriam ser uma só:
1. Os **SLOs** do runbook.
2. Os **limites** dos alertas.
3. Os **nomes das métricas** nas consultas (runbook e alertas usam nomes diferentes entre si).

Além disso, o endpoint `/metrics` da aplicação devolve um JSON de status, não métricas no formato do Prometheus. Ver [capítulo 14](14-o-que-a-ia-deixou-passar.md).

Próximo: [14 — O que a IA deixou passar](14-o-que-a-ia-deixou-passar.md)
