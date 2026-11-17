# Github-Action-Dotnet-MyDemo

A minimal ASP.NET Core (.NET 8) app deployed to Azure App Service by GitHub Actions.
The `/` endpoint returns `Hello World!`.

## Run locally

Install the .NET 8 SDK, then run these commands from the repository root:

```sh
dotnet restore
dotnet build --configuration Release --no-restore
dotnet test --configuration Release --no-build
dotnet run --configuration Release --no-build --urls http://localhost:5000
```

Open http://localhost:5000. There is no test project yet; `dotnet test` is included
in the workflow so future tests will run and failures will stop deployment.
To produce the deployment package:

```sh
dotnet publish --configuration Release --no-build --output ./publish
```

## Configure Azure deployment

The workflow is in `.github/workflows/azure-webapp.yml` (GitHub requires the
plural `workflows` directory). It runs on pushes to `main` or manually through
the Actions tab.

1. In Azure, create a Microsoft Entra application/service principal or a
   user-assigned managed identity for GitHub's OpenID Connect (OIDC) login.
   Grant it **Contributor** at subscription scope if the workflow should create
   the resource group. Alternatively, pre-create `rg-az400-demo` and grant
   Contributor only on that resource group.
2. Add a federated credential with issuer `https://token.actions.githubusercontent.com`,
   audience `api://AzureADTokenExchange`, and subject
   `repo:alodha100/Github-Action-Dotnet-MyDemo:environment:production`.
   The environment subject is required because the deployment job uses
   `production`, even when triggered from `main`.
3. Create the GitHub environment **production** under Settings > Environments.
   Optionally add required reviewers for an approval demo and restrict deployment
   branches to `main`.
4. Add these repository or production-environment Actions secrets:
   - `AZURE_CLIENT_ID`: the application or managed identity client ID.
   - `AZURE_TENANT_ID`: the Microsoft Entra tenant ID.
   - `AZURE_SUBSCRIPTION_ID`: the target Azure subscription ID.
   No client secret or publish profile is needed.
5. Review the workflow's `env` values. Set `WEBAPP_NAME` to a globally unique
   name; adjust the resource group, region, and plan as needed.
6. Push to `main` or select **Run workflow** in Actions. Approve the production
   deployment if required reviewers are configured.

The workflow builds and publishes the app, transfers it as an artifact, creates
the resource group, Linux B1 plan, and .NET 8 web app if needed, and deploys to
`https://<WEBAPP_NAME>.azurewebsites.net`. Re-runs reuse resources with the same
names. The artifact download action uses a patched v4 release, and test failures
are not ignored.

**Cost:** the B1 App Service plan incurs Azure charges. After the demo, delete
the demo resource group if it contains no resources you need:

```sh
az group delete --name rg-az400-demo --yes --no-wait
```