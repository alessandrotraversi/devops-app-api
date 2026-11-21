# 11 — Docker Compose e infra local

## O que é?

O **Docker Compose** sobe vários containers juntos, descritos num único arquivo, numa rede compartilhada. É o "Kubernetes de bolso" para desenvolvimento.

O app-api tem **três formas** de rodar localmente — e entender a diferença é parte do aprendizado:

| Forma | O que sobe | Para quê |
|---|---|---|
| `infra-local.sh` | Só Postgres e Redis (via `docker run`) | Rodar a API direto no seu Python, com debugger |
| `docker-compose.yml` | API + Postgres + Redis + coletor OTel | Ambiente completo com 1 comando |
| `k8s/` + k3d | Tudo, num Kubernetes local | Testar o que vai para "produção" |

## Mapa de conceitos: Compose ↔ Kubernetes

| No Compose | No Kubernetes | Ideia |
|---|---|---|
| `services:` | Deployment / StatefulSet + Service | Um componente que roda |
| `image:` / `build:` | `image:` | O pacote |
| `environment:` | ConfigMap / Secret | Configuração |
| `ports: "8000:8000"` | Service + Ingress | Expor para fora |
| `volumes:` (nomeado) | PVC | Dados que sobrevivem |
| `healthcheck:` | probes | "Você está bem?" |
| `depends_on: condition: service_healthy` | (não existe igual) | Ordem de inicialização |
| `networks:` | rede do cluster + NetworkPolicy | Quem fala com quem |

## Conceitos do Compose do app-api

- **`depends_on` com `service_healthy`**: a API só sobe quando o Postgres e o Redis passam no healthcheck. No Kubernetes não existe isso — lá a aplicação precisa **tolerar** a dependência estar fora do ar (é para isso que serve a readiness).
- **Bind mount do código + `--reload`**: a pasta do código do seu computador é "espelhada" dentro do container e o servidor reinicia a cada alteração. Ótimo para dev, nunca para produção.
- **DNS por nome do serviço**: a API acessa o banco pelo nome `postgres` — mesmo conceito do Service no Kubernetes.
- **Volume nomeado** para os dados do Postgres: `docker compose down` não apaga; `docker compose down -v` apaga.

## Conceitos do `infra-local.sh`

Um script Bash com 3 subcomandos (`up`, `down`, `status`). Destaques:

- `set -e` → o script para no primeiro erro.
- **Laço de espera** (`until ... pg_isready`) → espera ativa até o banco aceitar conexões. É o mesmo conceito de uma readiness probe, feito à mão.

## Pontos de atenção

- O endereço do coletor OTel no Compose é um **IP fixo** da rede Docker, em vez do nome do serviço (`otel-collector`). IPs de rede Docker mudam — o nome não.
- A chave `version:` no topo do Compose é obsoleta (as versões atuais ignoram).
- Credenciais escritas direto no arquivo. Para dev local é tolerável, mas o hábito escorrega para os manifestos de cluster (como de fato aconteceu).

## Comandos para explorar

```bash
docker compose up -d
docker compose ps           # veja o status de saúde de cada serviço
docker compose logs -f app
docker compose down         # mantém os dados
docker compose down -v      # apaga os volumes também
```

Próximo: [12 — Pipelines](12-pipelines.md)
