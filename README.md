# Power Platform pipeline: Dev → UAT → Prod with GitHub Actions

- **`dev-para-uat.yml`** ("Dev to UAT", on push to `main` or manual): exports the solution from Dev (unmanaged + managed), unpacks it into `solutions/<solution-name>/`, commits it, repacks it as a sanity check and imports the managed package into UAT. The commit is made with `GITHUB_TOKEN`, which does not trigger new workflow runs, so there is no loop.
- **`deploy-prod.yml`** ("Deploy Prod", manual only): asks for the confirmation word `PRODUCAO`, downloads the managed package from the latest successful UAT run (the same package that was tested) and imports it into Prod. If there is no such run, it repacks from `solutions/`.

Set the repository variable `PP_SOLUTION_NAME` to the solution's **unique** name.

## 1. Service principal (one time)

**Option A, with Azure CLI:**
```bash
az login
APP_ID=$(az ad app create --display-name "sp-powerplatform-github" --query appId -o tsv)
az ad sp create --id "$APP_ID"
az ad app credential reset --id "$APP_ID" --display-name "github-actions" --years 1 --query password -o tsv   # = PAC_CLIENT_SECRET (store it safely!)
az account show --query tenantId -o tsv                                                                       # = PAC_TENANT_ID
echo "$APP_ID"                                                                                                # = PAC_CLIENT_ID
```

**Option B, with pac (creates the Entra ID app and the Dev application user in one go; shows the client secret on screen):**
```bash
pac auth create                      # interactive sign-in with an admin
pac admin create-service-principal --environment <DEV_URL>
```

**Register it as an application user in each environment** (Dev, UAT and Prod; with option B, Dev is already done). You can also do this in the Power Platform admin center → Environment → Settings → Users + permissions → Application users.
```bash
pac admin assign-user --environment <DEV_URL>  --user <APP_ID> --role "System Administrator" --application-user
pac admin assign-user --environment <UAT_URL>  --user <APP_ID> --role "System Administrator" --application-user
pac admin assign-user --environment <PROD_URL> --user <APP_ID> --role "System Administrator" --application-user
```

## 2. GitHub: environments, secrets and variables

Create the environments `dev`, `uat` and `production` (Settings → Environments). To get an approval step before Prod, enable **Required reviewers** on `production`.

Go to **Settings → Secrets and variables → Actions** and create:

| Name | Type | Value |
|---|---|---|
| `PAC_CLIENT_SECRET` | **Secret** | client secret value of the app registration |
| `PAC_CLIENT_ID` | Variable | Application (client) ID |
| `PAC_TENANT_ID` | Variable | Directory (tenant) ID |
| `PP_DEV_URL` | Variable | Dev environment URL (e.g. `https://org-dev.crm2.dynamics.com`) |
| `PP_UAT_URL` | Variable | UAT environment URL |
| `PP_PROD_URL` | Variable | Prod environment URL |
| `PP_SOLUTION_NAME` | Variable | solution unique name |

The workflows read `PAC_CLIENT_ID` and `PAC_TENANT_ID` from variables first and fall back to secrets with the same name. **Never store the client secret as a variable:** variable values are printed in plain text in the run logs, and the logs of a public repo are public.

```bash
REPO=<OWNER>/<REPO>
for ENV in dev uat production; do gh api --method PUT repos/$REPO/environments/$ENV; done
gh secret set PAC_CLIENT_SECRET --repo $REPO   # paste the value when prompted (it is not saved in shell history)
gh variable set PAC_CLIENT_ID    --repo $REPO --body "<APP_ID>"
gh variable set PAC_TENANT_ID    --repo $REPO --body "<TENANT_ID>"
gh variable set PP_DEV_URL       --repo $REPO --body "<DEV_URL>"
gh variable set PP_UAT_URL       --repo $REPO --body "<UAT_URL>"
gh variable set PP_PROD_URL      --repo $REPO --body "<PROD_URL>"
gh variable set PP_SOLUTION_NAME --repo $REPO --body "<SOLUTION_NAME>"
```

**Connection references:** the connections must exist in the target environment. If the solution has connection references or environment variables, generate a settings file with `pac solution create-settings --solution-zip <zip> --settings-file <file>.json` and pass `--settings-file` to `pac solution import`.

**Public repo:** run logs are public. Secrets are masked, but never `echo` sensitive values.

## 3. Demo script (3 steps)

1. **Push → green UAT:** make a change to the solution in Dev and push (or merge) to `main`. Show "Dev to UAT" exporting, committing to `solutions/` and importing into UAT.
2. **Manual Prod deploy:** in Actions → "Deploy Prod" → *Run workflow*, type `PRODUCAO`. If Required reviewers is enabled on `production`, this is where the approval happens.
3. **Import completed in Prod:** follow the job, then show the ✅ summary and the solution, with the new version, in make.powerapps.com in Prod.

**Backup plan:** run everything ahead of time and, if the live demo fails, show the completed runs (logs, the `pacote-managed` artifact and the summary).

test
