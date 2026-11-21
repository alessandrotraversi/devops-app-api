# 08 — NetworkPolicy

## O que é?

Por padrão, **todo pod pode falar com todo pod** do cluster, em qualquer namespace. Uma **NetworkPolicy** é um firewall que restringe isso.

> **Analogia:** crachás de acesso num escritório. Sem regra, todo mundo entra em qualquer sala. Com regra, só quem tem crachá do financeiro entra no financeiro.

Dois sentidos:
- **Ingress** (entrada): quem pode **chegar** até o pod.
- **Egress** (saída): para onde o pod pode **ir**.

> Não confunda com o recurso Ingress do capítulo 04. Mesmo nome, coisas diferentes.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: so-o-ingress-fala-com-a-api
spec:
  podSelector:
    matchLabels: { app: minha-api }   # a quem a regra se aplica
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 8000
```

## Como as regras funcionam (o ponto mais confuso)

1. Se **nenhuma** política seleciona um pod → tudo liberado.
2. Se **alguma** política seleciona o pod para um sentido (ingress/egress) → **tudo é bloqueado naquele sentido**, exceto o que alguma regra liberar.
3. As regras **somam** (são "listas brancas"). Não existe regra de "negar".

E sobre os seletores:

| Seletor | Significa |
|---|---|
| `namespaceSelector: {}` (vazio) | **Todos** os namespaces |
| `podSelector: {}` (vazio) | Todos os pods **do mesmo namespace** |
| `ipBlock` | Um intervalo de IPs (para coisas fora do cluster) |

## Como aparece no app-api

A política se aplica aos pods da API, nos dois sentidos:

- **Entrada**: libera a porta 8000 vinda de **qualquer namespace** (seletor vazio).
- **Saída**: libera ir para **qualquer namespace** e qualquer pod do próprio namespace.

Na prática: a política é quase "tudo liberado dentro do cluster". O único efeito real é **bloquear a saída para fora do cluster** (internet) — ninguém liberou um `ipBlock`.

⚠️ É uma política que "parece" segura mas protege pouco. E existe um detalhe: **NetworkPolicy só funciona se o plugin de rede do cluster suportar** (o k3s suporta; alguns ambientes não, e a política é ignorada em silêncio). Ver [capítulo 14](14-o-que-a-ia-deixou-passar.md).

## Não esqueça do DNS

Ao restringir saída (egress), lembre de liberar o DNS (porta 53 UDP/TCP para o `kube-system`). Sem isso o pod não resolve o nome `meu-banco` e tudo quebra com erros confusos.

## Comandos para explorar

```bash
kubectl get networkpolicy -n meu-projeto
kubectl describe networkpolicy <nome>
# Testar conectividade de dentro de um pod temporário:
kubectl run teste --rm -it --image=busybox -- wget -T 3 -O- http://minha-api:8000/health
```

Próximo: [09 — Alertas](09-alertas.md)
