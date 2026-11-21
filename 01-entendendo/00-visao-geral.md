# 00 — Visão geral: o mapa

Antes de olhar arquivo por arquivo, vale entender **o que o Kubernetes é** e **como as peças do app-api se encaixam**.

## Kubernetes em uma frase

Você descreve **como quer que as coisas estejam** ("quero 2 cópias da minha API rodando") e o Kubernetes fica, sem parar, comparando isso com **como as coisas estão** — e corrigindo a diferença.

Isso se chama **modelo declarativo**. Você não diz "inicie um container"; você diz "deve existir um container". Se ele morrer, o Kubernetes cria outro sozinho.

> **Analogia:** é como um termostato. Você não liga e desliga o ar-condicionado; você diz "quero 22 °C" e ele se vira.

## Todo arquivo YAML do Kubernetes tem a mesma "cara"

```yaml
apiVersion: ...   # qual "versão do dicionário" o Kubernetes deve usar
kind: ...         # que tipo de coisa é (Deployment, Service, ...)
metadata:         # nome, namespace e etiquetas (labels)
  name: ...
spec:             # o "desejo": como você quer que isso fique
  ...
```

Se você entender essas 4 partes, já consegue ler qualquer manifesto. O que muda de um tipo para outro é só o conteúdo do `spec`.

## O mapa do app-api

```
                 Internet / seu navegador
                           │
                           ▼
                ┌────────────────────┐
                │      Ingress       │  "porteiro": recebe pelo nome do site
                └─────────┬──────────┘
                          ▼
                ┌────────────────────┐
                │  Service (da API)  │  "telefone fixo": endereço estável
                └─────────┬──────────┘
                          ▼
     ┌──────────── Deployment (API) ─────────────┐ ◄── HPA: aumenta/diminui
     │   ┌───────┐   ┌───────┐   ...até 10       │      o número de pods
     │   │  Pod  │   │  Pod  │                   │
     │   └───┬───┘   └───┬───┘                   │ ◄── ConfigMap + Secret:
     └───────┼───────────┼───────────────────────┘      configurações e senhas
             │           │
     ┌───────┴───┐   ┌───┴────────┐
     ▼           │   │            ▼
 Service       ...   ...       Service
 (Postgres)                    (Redis)
     │                            │
     ▼                            ▼
 StatefulSet                  StatefulSet
 Postgres + disco             Redis (cache)

 Tudo isso vive dentro de um Namespace.
 Uma NetworkPolicy define quem pode conversar com a API.
 Uma PrometheusRule define quando disparar alertas.
```

## Os 3 "andares" do projeto

| Andar | Pergunta que responde | Arquivos |
|---|---|---|
| **Empacotar** | Como transformo meu código num pacote que roda em qualquer lugar? | `Dockerfile` |
| **Rodar** | Onde e como esse pacote roda, conversa e escala? | `docker-compose.yml` (local), `k8s/*.yaml` (cluster) |
| **Entregar e vigiar** | Como o código chega lá automaticamente e como sei se está saudável? | `.github/workflows/*`, `otel-collector-config.yaml`, `alert-rules.yaml`, runbook |

## Labels: a "cola" de tudo

Uma coisa que confunde no começo: **como o Service sabe para quais Pods mandar tráfego?** Não é pelo nome. É por **etiquetas** (labels).

- O Pod é criado com a etiqueta `app: minha-api`.
- O Service diz: "mande tráfego para quem tiver a etiqueta `app: minha-api`".
- O HPA, a NetworkPolicy e as consultas de alerta também usam etiquetas ou nomes.

> Se uma etiqueta estiver escrita diferente em dois lugares, **nada dá erro** — simplesmente não conecta. É o bug mais silencioso do Kubernetes.

Próximo: [01 — Namespace](01-namespace.md)
