# 01 — Evoluindo: Segredos

**Onde estamos:** senha em base64 commitada no Git, e repetida em texto puro dentro de uma URL no ConfigMap.

---

## 🥉 Nível 1 — Tirar do Git

1. **Nunca monte URLs com senha no ConfigMap.** Deixe no ConfigMap só as partes não sensíveis (host, porta, nome do banco) e monte a URL **na aplicação**, lendo a senha de uma variável vinda do Secret.
   Alternativa: o Kubernetes expande variáveis já definidas no mesmo container:
   ```yaml
   env:
     - name: DB_PASSWORD
       valueFrom: { secretKeyRef: { name: segredos-da-api, key: DB_PASSWORD } }
     - name: DATABASE_URL
       value: "postgresql://app:$(DB_PASSWORD)@meu-banco:5432/app"
   ```
2. **Crie o Secret fora do Git**, direto no cluster:
   ```bash
   kubectl create secret generic segredos-da-api \
     --from-literal=DB_PASSWORD="$(openssl rand -base64 24)"
   ```
3. Apague o arquivo do Secret do repositório, coloque-o no `.gitignore` e **troque a senha** (ela continua no histórico do Git).

**Limitação:** agora o Secret não está versionado em lugar nenhum. Se o cluster sumir, a senha some.

---

## 🥈 Nível 2 — Segredo criptografado no Git

Ferramentas que permitem commitar o segredo **criptografado**:

| Ferramenta | Como funciona |
|---|---|
| **Sealed Secrets** | Você criptografa com a chave pública do cluster; só o cluster consegue abrir. Gera um recurso `SealedSecret` que pode ir para o Git |
| **SOPS** (+ age ou KMS) | Criptografa só os *valores* do YAML. Integra com Kustomize/Argo CD/Flux |

Fluxo com Sealed Secrets:
```bash
kubectl create secret generic segredos-da-api --from-literal=DB_PASSWORD=xyz \
  --dry-run=client -o yaml | kubeseal -o yaml > sealed-secret.yaml
git add sealed-secret.yaml   # seguro: só o cluster decifra
```

---

## 🥇 Nível 3 — Cofre central

O segredo mora num **cofre** (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault). O **External Secrets Operator** sincroniza do cofre para um Secret do Kubernetes.

Ganhos: rotação automática, auditoria (quem leu, quando), um único lugar para várias aplicações e clusters. Ative também a **criptografia em repouso** do etcd.

---

## Como validar

```bash
git log -p | grep -i password          # o histórico ainda tem senhas?
kubectl get configmap -A -o yaml | grep -i pass
trivy fs --scanners secret .           # detector de segredos
```
