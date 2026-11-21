# 01 — Namespace

## O que é?

Um **Namespace** é uma "pasta" dentro do cluster. Ele separa recursos de projetos (ou ambientes) diferentes para que não se misturem.

> **Analogia:** um prédio (o cluster) com vários apartamentos (namespaces). Cada apartamento pode ter um "quarto" com o mesmo nome sem conflito.

Exemplo mínimo:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: meu-projeto
```

## Como aparece no app-api

Existe um namespace próprio para a aplicação. **Todos** os outros manifestos dizem em `metadata.namespace` que pertencem a ele. Ele também recebe uma label com o próprio nome — útil para outras regras (por exemplo, NetworkPolicy) selecionarem esse namespace.

## O que o Namespace faz e o que **não** faz

| Faz | Não faz (sozinho) |
|---|---|
| Separa nomes (dois Services `api` em namespaces diferentes) | Não isola a rede — por padrão, pods de namespaces diferentes se falam |
| Permite dar permissões por namespace (RBAC) | Não limita CPU/memória — para isso existe `ResourceQuota` |
| Facilita apagar tudo de uma vez (`kubectl delete ns ...`) | Não protege contra um pod "comer" o nó inteiro |

## Pontos de atenção

- **Ordem de criação:** o namespace precisa existir antes dos outros recursos. Com `kubectl apply -f k8s/` os arquivos são aplicados em ordem alfabética — `namespace.yaml` não é o primeiro! Funciona na segunda execução, mas pode falhar na primeira. (Veja como resolver em [02-evoluindo](../02-evoluindo/03-empacotando-manifestos.md).)
- **DNS:** dentro do cluster, um Service pode ser chamado por `nome` (mesmo namespace) ou `nome.namespace.svc.cluster.local` (de qualquer lugar).

## Comandos para explorar

```bash
kubectl get namespaces
kubectl get all -n meu-projeto
kubectl config set-context --current --namespace=meu-projeto   # deixa como padrão
```

Próximo: [02 — Deployment](02-deployment.md)
