# 10 — Dockerfile

## O que é?

A "receita" para construir a **imagem**: um pacote com sistema operacional mínimo + linguagem + dependências + seu código. A imagem é o que o Kubernetes roda.

> **Analogia:** a receita (Dockerfile) gera um bolo congelado (imagem). Qualquer forno (servidor) assa igual.

Cada instrução cria uma **camada**. Camadas que não mudaram são reaproveitadas do **cache**, o que acelera o build.

## Conceito 1: multi-stage build

O app-api usa **dois estágios**:

```dockerfile
# Estágio 1: "oficina" — tem ferramentas de build
FROM python:3.12-slim AS builder
RUN python -m venv /venv && /venv/bin/pip install -r requirements.txt

# Estágio 2: "vitrine" — só o necessário para rodar
FROM python:3.12-slim
COPY --from=builder /venv /venv
COPY app/ /app/
```

A imagem final só leva o que foi copiado explicitamente. Ferramentas de compilação ficam para trás → imagem **menor** e com **menos vulnerabilidades**.

## Conceito 2: ordem das instruções = velocidade

O app-api copia `requirements.txt` e instala dependências **antes** de copiar o código. Por quê? Você muda o código toda hora, mas as dependências raramente. Assim, a camada pesada (`pip install`) vem do cache na maioria dos builds.

## Conceito 3: usuário não-root

O Dockerfile cria um usuário com **ID 1000**, dá a ele a posse da pasta da aplicação e troca para ele (`USER`). Isso conversa diretamente com o `securityContext` do Deployment (cap. 02), que exige não-root e ID 1000. **Os dois lados precisam combinar.**

## Conceito 4: variáveis de ambiente

- `PYTHONDONTWRITEBYTECODE` → não grava arquivos `.pyc` (combina com disco somente-leitura).
- `PYTHONUNBUFFERED` → os logs saem na hora, sem ficar presos em buffer (essencial para `kubectl logs`).
- Variáveis `OTEL_*` → configuram o envio de telemetria (cap. 13).

⚠️ O app-api "assa" na imagem valores de ambiente como `deployment.environment=dev` e o endereço do coletor. Isso amarra a imagem a um ambiente — o oposto do conceito do cap. 05. O ideal é deixar isso no ConfigMap.

## Conceito 5: HEALTHCHECK

O Dockerfile define um `HEALTHCHECK` com `curl`. Ele é usado pelo **Docker** e pelo **Compose**, mas o **Kubernetes ignora** — lá quem manda são as probes. Por isso o `curl` foi instalado na imagem final, adicionando tamanho e superfície de ataque só para algo que o cluster não usa.

## Conceito 6: CMD

O comando final inicia o servidor web (uvicorn) escutando em `0.0.0.0` (todas as interfaces — necessário dentro de container; `127.0.0.1` só seria acessível de dentro dele mesmo).

## Pontos de atenção (resumo)

- No estágio de build também se instala `curl`, que não é usado lá.
- Não existe `.dockerignore` — tudo do diretório vai no contexto de build (inclusive `.git`, testes...).
- `python:3.11-slim` sem versão fixa de patch/digest → o build de hoje pode ser diferente do de amanhã.

## Comandos para explorar

```bash
docker build -t minha-api:dev .
docker history minha-api:dev          # veja as camadas e o tamanho de cada uma
docker run --rm -p 8000:8000 minha-api:dev
docker run --rm minha-api:dev id      # confirme que não é root
```

Próximo: [11 — Docker Compose](11-docker-compose.md)
