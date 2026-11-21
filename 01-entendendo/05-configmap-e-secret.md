# 05 — ConfigMap e Secret

## O que são?

São "dicionários" de chave/valor guardados no cluster, que os pods consomem como **variáveis de ambiente** ou **arquivos**.

- **ConfigMap** → configuração comum (host do banco, porta, nível de log).
- **Secret** → dado sensível (senha, token, chave).

> **Analogia:** o ConfigMap é o quadro de avisos da cozinha; o Secret é o cofre. Os dois ficam na mesma casa, mas com regras de acesso diferentes.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-da-api
data:
  LOG_LEVEL: "info"
  DB_HOST: "meu-banco"
---
apiVersion: v1
kind: Secret
metadata:
  name: segredos-da-api
type: Opaque
stringData:          # texto normal; o Kubernetes converte para base64
  DB_PASSWORD: "troque-me"
```

## Por que separar a config da imagem?

Princípio dos **12 fatores**: o mesmo artefato (imagem) deve rodar em dev, homologação e produção. O que muda entre eles é só a **configuração**, injetada de fora.

## A grande pegadinha: base64 **não** é criptografia

No campo `data` de um Secret os valores ficam em **base64**. Muita gente (e muita IA) trata isso como se fosse "escondido". Não é:

```bash
echo 'c2VuaGExMjM=' | base64 -d     # → senha123
```

Qualquer pessoa que lê o arquivo lê a senha. Ou seja: **um Secret commitado no Git é uma senha pública**.

O que o Secret tem de diferente do ConfigMap:
- Pode ter permissões (RBAC) mais restritas.
- Pode ser criptografado *no banco do cluster* (etcd), se configurado.
- Não aparece em `kubectl describe`.

O que ele **não** resolve: guardar o valor no repositório.

## Como aparece no app-api

- O ConfigMap guarda hosts e portas do Postgres e do Redis, e URLs de conexão completas.
- O Secret guarda usuário e senha do banco, em base64, **commitado no repositório**.
- O Deployment da API importa os dois inteiros (`envFrom`). O StatefulSet do Postgres puxa chaves específicas do Secret (`secretKeyRef`).

⚠️ Uma das URLs no ConfigMap contém usuário **e senha** embutidos — ou seja, a senha "protegida" no Secret também está em texto puro no ConfigMap. Ver [capítulo 14](14-o-que-a-ia-deixou-passar.md).

## Duas formas de consumir

| Forma | Exemplo | Observação |
|---|---|---|
| Tudo de uma vez | `envFrom: - secretRef: ...` | Simples, mas importa chaves que talvez não precise |
| Chave a chave | `env: - name: X, valueFrom: secretKeyRef: ...` | Mais explícito |
| Como arquivo | `volumes: - secret: ...` | Bom para certificados; atualiza sem reiniciar |

> Mudar um ConfigMap **não** reinicia os pods. Variáveis de ambiente só são lidas quando o container nasce. É preciso `kubectl rollout restart`.

## Comandos para explorar

```bash
kubectl get configmap config-da-api -o yaml
kubectl get secret segredos-da-api -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
kubectl create secret generic segredos-da-api --from-literal=DB_PASSWORD=xyz --dry-run=client -o yaml
```

O último comando é útil: gera o YAML sem você precisar fazer base64 na mão.

Próximo: [06 — StatefulSet](06-statefulset.md)
