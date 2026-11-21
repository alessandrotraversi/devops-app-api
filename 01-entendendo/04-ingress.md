# 04 — Ingress

## O que é?

O **Ingress** é a porta de entrada HTTP do cluster. Ele olha o **domínio** e o **caminho** da requisição e decide para qual Service mandar.

> **Analogia:** a recepção de um prédio comercial. "Veio para a empresa X? 5º andar. Para a Y? 8º andar."

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minha-api
spec:
  rules:
    - host: api.exemplo.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: minha-api
                port:
                  number: 80
```

## Ingress vs. Ingress Controller

O Ingress é só uma **regra escrita**. Quem lê a regra e realmente recebe o tráfego é um programa chamado **Ingress Controller** (NGINX, Traefik, HAProxy...). Sem um controller instalado, o Ingress não faz nada.

O **k3d/k3s já vem com o Traefik** instalado. Por isso o app-api tem uma *annotation* específica do Traefik, dizendo para usar a entrada `web` (HTTP, porta 80).

> **Annotations** são "bilhetes" para ferramentas específicas. O Kubernetes em si ignora; o controller lê.

## Como aparece no app-api

- Um domínio fictício terminado em `.test` (reservado para testes, nunca existe na internet). Para funcionar no seu computador, você aponta esse nome para `127.0.0.1` no arquivo `/etc/hosts`.
- Um único caminho (`/`) do tipo `Prefix` = "tudo que começar com `/`", ou seja, tudo.
- Sem HTTPS (só a entrada `web`).

## `pathType`

| Valor | Comportamento |
|---|---|
| `Prefix` | `/api` casa com `/api`, `/api/`, `/api/items` |
| `Exact` | Só casa com exatamente `/api` |
| `ImplementationSpecific` | Depende do controller (evite) |

## Pontos de atenção

- O Ingress aponta para um Service **pelo nome**. Se o nome estiver errado, o controller devolve 404/503 e nada no `kubectl apply` reclama.
- O campo moderno para escolher o controller é `spec.ingressClassName`. O app-api depende do controller padrão do cluster.
- Sem TLS, tudo trafega em texto puro. Ver [02-evoluindo/05](../02-evoluindo/05-seguranca.md).

## Comandos para explorar

```bash
kubectl get ingress -n meu-projeto
kubectl describe ingress minha-api -n meu-projeto
curl -H "Host: api.exemplo.test" http://localhost/   # testa sem mexer no /etc/hosts
```

Próximo: [05 — ConfigMap e Secret](05-configmap-e-secret.md)
