# Pipeline Power Platform: Dev → UAT → Prod com GitHub Actions

- **`dev-para-uat.yml`** (push no `main` ou manual): exporta a solução do Dev (unmanaged + managed), faz unpack em `solutions/<NOME_DA_SOLUCAO>/`, commita, reempacota como checagem básica e importa o managed no UAT. O commit é feito com `GITHUB_TOKEN`, que não dispara novos workflows, então não há loop.
- **`deploy-prod.yml`** (somente manual): pede a confirmação `PRODUCAO`, baixa o pacote managed da última execução bem-sucedida do UAT (o mesmo pacote testado) e importa no Prod. Se não houver execução, reempacota a partir de `solutions/`.

Troque `<NOME_DA_SOLUCAO>` pelo nome **único** da solução (ou crie a variável de repositório `PP_SOLUTION_NAME`).

## 1. Service principal (uma vez)

**Opção A, com Azure CLI:**
```bash
az login
APP_ID=$(az ad app create --display-name "sp-powerplatform-github" --query appId -o tsv)
az ad sp create --id "$APP_ID"
az ad app credential reset --id "$APP_ID" --display-name "github-actions" --years 1 --query password -o tsv   # = PAC_CLIENT_SECRET (guarde!)
az account show --query tenantId -o tsv                                                                       # = PAC_TENANT_ID
echo "$APP_ID"                                                                                                # = PAC_CLIENT_ID
```

**Opção B, com pac (cria o app no Entra ID e o usuário de aplicativo no Dev de uma vez; mostra o Client Secret na tela):**
```bash
pac auth create                      # login interativo com um admin
pac admin create-service-principal --environment <URL_DEV>
```

**Registrar como usuário de aplicativo em cada ambiente** (faça para Dev, UAT e Prod; na opção B o Dev já está feito). Você também pode fazer isso pelo Power Platform admin center → Ambiente → Configurações → Usuários + permissões → Usuários de aplicativo.
```bash
pac admin assign-user --environment <URL_DEV>  --user <APP_ID> --role "System Administrator" --application-user
pac admin assign-user --environment <URL_UAT>  --user <APP_ID> --role "System Administrator" --application-user
pac admin assign-user --environment <URL_PROD> --user <APP_ID> --role "System Administrator" --application-user
```

## 2. GitHub: environments, secrets e variáveis

Crie os environments `dev`, `uat` e `production`, cada um com os secrets `PAC_CLIENT_ID`, `PAC_CLIENT_SECRET` e `PAC_TENANT_ID`. Se for usar o mesmo SP em tudo, repita os mesmos valores.

As URLs dos ambientes não são sensíveis, então ficam em **variáveis de repositório**, não em secrets. Vá em **Settings → Secrets and variables → Actions → aba Variables → New repository variable** e crie:

| Variável | Valor |
|---|---|
| `PP_DEV_URL` | URL do ambiente Dev (ex.: `https://org-dev.crm2.dynamics.com`) |
| `PP_UAT_URL` | URL do ambiente UAT |
| `PP_PROD_URL` | URL do ambiente Prod |
| `PP_SOLUTION_NAME` | nome único da solução |
```bash
REPO=<OWNER>/<REPO>
gh variable set PP_SOLUTION_NAME --repo $REPO --body "<NOME_DA_SOLUCAO>"

for ENV in dev uat production; do
  gh api --method PUT repos/$REPO/environments/$ENV
  gh secret set PAC_CLIENT_ID     --repo $REPO --env $ENV --body "<APP_ID>"
  gh secret set PAC_CLIENT_SECRET --repo $REPO --env $ENV   # cole o valor quando pedir (não fica no histórico)
  gh secret set PAC_TENANT_ID     --repo $REPO --env $ENV --body "<TENANT_ID>"
done
gh variable set PP_DEV_URL  --repo $REPO --body "<URL_DEV>"
gh variable set PP_UAT_URL  --repo $REPO --body "<URL_UAT>"
gh variable set PP_PROD_URL --repo $REPO --body "<URL_PROD>"
```

**Connection references:** as conexões precisam existir no ambiente de destino. Se a solução tiver connection references ou variáveis de ambiente, gere um settings file com `pac solution create-settings --solution-zip <zip> --settings-file <arquivo>.json` e passe `--settings-file` no `pac solution import`.

**Repo público:** os logs das execuções são públicos. Os secrets ficam mascarados, mas nunca dê `echo` em valores sensíveis.

## 3. Roteiro da demo (3 passos)

1. **Push → UAT verde:** faça uma alteração na solução no Dev e um push (ou merge) no `main`. Mostre o "Dev para UAT" exportando, commitando em `solutions/` e importando no UAT.
2. **Deploy Prod manual:** em Actions → "Deploy Prod" → *Run workflow*, digite `PRODUCAO`. Esse é o momento da aprovação.
3. **Import concluído no Prod:** acompanhe o job e mostre o resumo ✅ e a solução, com a versão nova, em make.powerapps.com no Prod.

**Plano B:** rode tudo hoje à noite e, se a demo ao vivo falhar, mostre essas execuções já concluídas (logs, artefato `pacote-managed` e resumo).
