# Pipeline Power Platform: Dev â†’ UAT â†’ Prod com GitHub Actions

- **`dev-para-uat.yml`** (push no `main` ou manual): exporta a soluÃ§Ã£o do Dev (unmanaged + managed), faz unpack em `solutions/<NOME_DA_SOLUCAO>/`, commita, reempacota como checagem bÃ¡sica e importa o managed no UAT. O commit Ã© feito com `GITHUB_TOKEN`, que nÃ£o dispara novos workflows, entÃ£o nÃ£o hÃ¡ loop.
- **`deploy-prod.yml`** (somente manual): pede a confirmaÃ§Ã£o `PRODUCAO`, baixa o pacote managed da Ãºltima execuÃ§Ã£o bem-sucedida do UAT (o mesmo pacote testado) e importa no Prod. Se nÃ£o houver execuÃ§Ã£o, reempacota a partir de `solutions/`.

Troque `<NOME_DA_SOLUCAO>` pelo nome **Ãºnico** da soluÃ§Ã£o (ou crie a variÃ¡vel de repositÃ³rio `PP_SOLUTION_NAME`).

## 1. Service principal (uma vez)

**OpÃ§Ã£o A, com Azure CLI:**
```bash
az login
APP_ID=$(az ad app create --display-name "sp-powerplatform-github" --query appId -o tsv)
az ad sp create --id "$APP_ID"
az ad app credential reset --id "$APP_ID" --display-name "github-actions" --years 1 --query password -o tsv   # = PAC_CLIENT_SECRET (guarde!)
az account show --query tenantId -o tsv                                                                       # = PAC_TENANT_ID
echo "$APP_ID"                                                                                                # = PAC_CLIENT_ID
```

**OpÃ§Ã£o B, com pac (cria o app no Entra ID e o usuÃ¡rio de aplicativo no Dev de uma vez; mostra o Client Secret na tela):**
```bash
pac auth create                      # login interativo com um admin
pac admin create-service-principal --environment <URL_DEV>
```

**Registrar como usuÃ¡rio de aplicativo em cada ambiente** (faÃ§a para Dev, UAT e Prod; na opÃ§Ã£o B o Dev jÃ¡ estÃ¡ feito). VocÃª tambÃ©m pode fazer isso pelo Power Platform admin center â†’ Ambiente â†’ ConfiguraÃ§Ãµes â†’ UsuÃ¡rios + permissÃµes â†’ UsuÃ¡rios de aplicativo.
```bash
pac admin assign-user --environment <URL_DEV>  --user <APP_ID> --role "System Administrator" --application-user
pac admin assign-user --environment <URL_UAT>  --user <APP_ID> --role "System Administrator" --application-user
pac admin assign-user --environment <URL_PROD> --user <APP_ID> --role "System Administrator" --application-user
```

## 2. GitHub: environments, secrets e variÃ¡veis

Crie os environments `dev`, `uat` e `production`, cada um com os secrets `PAC_CLIENT_ID`, `PAC_CLIENT_SECRET` e `PAC_TENANT_ID`. Se for usar o mesmo SP em tudo, repita os mesmos valores.

As URLs dos ambientes nÃ£o sÃ£o sensÃ­veis, entÃ£o ficam em **variÃ¡veis de repositÃ³rio**, nÃ£o em secrets. VÃ¡ em **Settings â†’ Secrets and variables â†’ Actions â†’ aba Variables â†’ New repository variable** e crie:

| VariÃ¡vel | Valor |
|---|---|
| `PP_DEV_URL` | URL do ambiente Dev (ex.: `https://org-dev.crm2.dynamics.com`) |
| `PP_UAT_URL` | URL do ambiente UAT |
| `PP_PROD_URL` | URL do ambiente Prod |
| `PP_SOLUTION_NAME` | nome Ãºnico da soluÃ§Ã£o |
```bash
REPO=<OWNER>/<REPO>
gh variable set PP_SOLUTION_NAME --repo $REPO --body "<NOME_DA_SOLUCAO>"

for ENV in dev uat production; do
  gh api --method PUT repos/$REPO/environments/$ENV
  gh secret set PAC_CLIENT_ID     --repo $REPO --env $ENV --body "<APP_ID>"
  gh secret set PAC_CLIENT_SECRET --repo $REPO --env $ENV   # cole o valor quando pedir (nÃ£o fica no histÃ³rico)
  gh secret set PAC_TENANT_ID     --repo $REPO --env $ENV --body "<TENANT_ID>"
done
gh variable set PP_DEV_URL  --repo $REPO --body "<URL_DEV>"
gh variable set PP_UAT_URL  --repo $REPO --body "<URL_UAT>"
gh variable set PP_PROD_URL --repo $REPO --body "<URL_PROD>"
```

**Connection references:** as conexÃµes precisam existir no ambiente de destino. Se a soluÃ§Ã£o tiver connection references ou variÃ¡veis de ambiente, gere um settings file com `pac solution create-settings --solution-zip <zip> --settings-file <arquivo>.json` e passe `--settings-file` no `pac solution import`.

**Repo pÃºblico:** os logs das execuÃ§Ãµes sÃ£o pÃºblicos. Os secrets ficam mascarados, mas nunca dÃª `echo` em valores sensÃ­veis.

## 3. Roteiro da demo (3 passos)

1. **Push â†’ UAT verde:** faÃ§a uma alteraÃ§Ã£o na soluÃ§Ã£o no Dev e um push (ou merge) no `main`. Mostre o "Dev para UAT" exportando, commitando em `solutions/` e importando no UAT.
2. **Deploy Prod manual:** em Actions â†’ "Deploy Prod" â†’ *Run workflow*, digite `PRODUCAO`. Esse Ã© o momento da aprovaÃ§Ã£o.
3. **Import concluÃ­do no Prod:** acompanhe o job e mostre o resumo âœ… e a soluÃ§Ã£o, com a versÃ£o nova, em make.powerapps.com no Prod.

**Plano B:** rode tudo hoje Ã  noite e, se a demo ao vivo falhar, mostre essas execuÃ§Ãµes jÃ¡ concluÃ­das (logs, artefato `pacote-managed` e resumo).
