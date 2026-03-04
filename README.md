# a11y-demo-frontend

Demo de **acessibilidade (a11y)** com *quality gate* em CI.

Este repositório contém um HTML de exemplo e um workflow de **GitHub Actions** que, a cada Pull Request, envia o HTML para uma **API serverless** (AWS API Gateway + Lambda + axe-core + DynamoDB + S3) e recebe um **score WCAG (0–100)**. Se o score ficar abaixo do **threshold** (ex.: 70), o job falha e pode bloquear o merge quando configurado como *required check*.

---

## Como funciona

1. O workflow roda em `pull_request` para a branch `main`.
2. Lê o arquivo `index.html` do repositório.
3. Chama a API `POST /score` enviando:

   * `repo`, `commit_sha` e `html`.
4. A API executa auditoria WCAG (axe-core), calcula o score e retorna:

   * `score`, `passed`, `violations_summary`, `report_s3_uri`, `timestamp` (e opcionalmente `previous_score`/`delta`).
5. Se `passed=false`, o workflow falha.

---

## Estrutura do repositório

* `index.html` — HTML de exemplo (pode conter violações propositalmente para testar o gate)
* `.github/workflows/a11y-gate.yml` — workflow do GitHub Actions

---

## Configuração do workflow

O workflow chama a API via variável `A11Y_API_BASE`.

Exemplo (base URL do stage `default`):

```text
https://912zbihl02.execute-api.sa-east-1.amazonaws.com/default
```

No YAML, a chamada fica assim:

```yaml
env:
  A11Y_API_BASE: https://912zbihl02.execute-api.sa-east-1.amazonaws.com/default
```

> Observação: no POC a API pode estar **Open** para facilitar testes. Em produção, a API deve ser protegida (IAM/JWT Authorizer).

---

## Como testar (manual)

### 1) Rodar o workflow

Abra um Pull Request para `main`.
O GitHub Actions executará o gate automaticamente.

### 2) Testar a API localmente (opcional)

Se a API estiver Open:

```bash
curl -X POST "https://912zbihl02.execute-api.sa-east-1.amazonaws.com/default/score" \
  -H "Content-Type: application/json" \
  -d '{
    "repo":"mmarleta/a11y-demo-frontend",
    "commit_sha":"manual-test",
    "html":"<!doctype html><html><body><img src=\"x.png\"><button></button></body></html>"
  }'
```

---

## Threshold / Gate

* Threshold padrão: **70**
* Se `score < threshold`, o job falha (`passed=false`).

Para bloquear merge no GitHub:

* Settings → Branches → Branch protection rules → marque o workflow como **Required status check**.

---

## Arquitetura do serviço (AWS)

* **API Gateway (HTTP API)** — expõe `POST /score` e `GET /score/{repo}/{commit_sha}`
* **Lambda (Node.js)** — executa auditoria com `axe-core` e calcula score
* **DynamoDB** — persiste histórico por `repo + commit_sha`
* **S3** — salva relatórios/evidências (ex.: `reports/<repo>/<commit>.json`)
* **CloudWatch Logs** — logs estruturados e troubleshooting
* **IAM** — permissões (ideal: least privilege)

Fluxo:

```text
PR (GitHub Actions)
  → API Gateway
    → Lambda (axe-core)
      → DynamoDB (scores)
      → S3 (reports/evidence)
  ← score/passed
```

---

## Roadmap (produção / “nível banco”)

O POC prioriza velocidade. Em produção, eu evoluiria para:

* **Autenticação/Autorização**:

  * JWT Authorizer (IdP corporativo: Azure AD/Okta/Cognito) e/ou IAM (SigV4)
* **Bypass governado com auditoria**:

  * Integração com Slack/ServiceNow (aprovação com justificativa + evidência + expiração/TTL)
* **Proteções adicionais**:

  * WAF, throttling/rate limits, KMS (SSE-KMS), lifecycle policies (artifacts vs reports)
* **Suporte a cenários mais completos**:

  * build estático (`dist/`) e/ou URL preview (Puppeteer/Lighthouse em execução assíncrona)

---

## Licença

Sem licença definida (demo).

