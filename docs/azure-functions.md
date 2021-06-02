# Azure Functions

## Include private python package as dependency of your Azure Functions app

Set the setting `PIP_EXTRA_INDEX_URL={URL_TO_YOUR_ARTIFACT}` in your Azure Functions app configuration (via the portal or VSCode extension). The URL_TO_YOUR_ARTIFACT will be of the following form: `https://{PAT}@pkgs.dev.azure.com/{organization}/_packaging/{artifactFeed}%40Local/pypi/simple/`.  You need to create a PAT via your account, see https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page. 

## Activate the cache on your Azure Functions app to optimize the deployment speed

Set the setting `WEBSITES_ENABLE_APP_CACHE=true` in your Azure Functions app configuration (via the portal or VSCode extension)
