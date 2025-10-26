# resume.stevebonds.com

My HTML resume content and DevOps automation to deploy it.

## Content

This is all static HTML and CSS, which allows for total control over the browser rendered appearance.

## Hosting

There are lots of options on making this content publicly available. One of the simplest is on GitHub itself. I discovered that this approach came with an unexpected side effect-- a massive drop in Google Search results. I switched over to hosting in Azure and the Google search results issue magically resolved.

Target URL: `https://resume.stevebonds.com`

This is all static content and requires no server-side changes, so hosting out of a simple Azure Storage Account is both inexpensive and simple.

Since I want it to run in a custom URL and use HTTPS, I also need to use an Azure Endpoint. To improve the end-user response time, I'm also using the Azure Front Door Content Distribution Network.

Currently, those resources are provisioned manually via the Azure Portal. I've only needed to do this once, so automating that portion of the process hasn't been a priority.

## Updates

I'm more familiar with Azure DevOps pipelines, so I'll do this using GitHub actions as a way to learn.

The general process follows [Microsoft Learn: Use GitHub Actions workflow to deploy your static website in Azure Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-static-site-github-actions?tabs=openid)

### Setting up access into Azure

I chose OpenID with User-assigned Managed Identity as the credential type since the Service Principal auth expires regularly, which is quite an annoyance for this rarely-used and long-lived pipeline.

I'll limit the scopes of the Managed Identity I create to a level which can upload content to a storage account and purge a CDN endpoint, in keeping with the security principle of least access. To make the command copy-paste-able, I'll set some configuration via environment variables which contain the Azure Resource ID for each of the the above resources:

```bash
export AZURE_STORAGE_ACCOUNT_ID=resource-id-for-the-web-content-storage-account
export AZURE_ENDPOINT_ID=resource-id-for-the-azure-endpoint
export AZURE_RESOURCE_GROUP_FOR_IDENTITY=resource-group-holding-above-resources
```

Create the Managed Identity using my Azure Owner account with the scope of the storage account and endpoint:

```bash
az identity create \
  --name "githubActionsForResumeOIDC" \
  --resource-group "$AZURE_RESOURCE_GROUP_FOR_IDENTITY"
```

The output was saved in my secure password storage.

Get the identity ID to use for role assignments since it doesn't seem to be found by name using `--assignee`:

```bash
identityId=$(az identity show --name "githubActionsForResumeOIDC" --resource-group "$AZURE_RESOURCE_GROUP_FOR_IDENTITY" --query principalId --out tsv)
```

Grant Contributor permissions to this Managed Identity, limited to the Storage Account and Azure Endpoint:

```bash
az role assignment create \
  --assignee $identityId \
  --role contributor \
  --scope "$AZURE_STORAGE_ACCOUNT_ID"
az role assignment create \
  --assignee $identityId \
  --role contributor \
  --scope "$AZURE_ENDPOINT_ID"
```

### Federating this identity with GitHub

I used the Azure Portal for this to assist with error checking.

TODO: Describe this process using Azure CLI.

### Providing the Managed Identity credential to GitHub

I'm using the GitHub web interface for simplicity, but this could be done more consistently and reproducibly using the GitHub CLI via `gh secret set`. The credential will be scoped to this repository only since this is the only place which needs access to those Azure resources.

The secrets created are:

* `AZURE_CLIENT_ID`: contents of `az identity show --name "githubActionsForResumeOIDC" --resource-group "$AZURE_RESOURCE_GROUP_FOR_IDENTITY" --query clientId`
* `AZURE_SUBSCRIPTION_ID`: `az account show --query id`
* `AZURE_TENANT_ID`: contents of `az identity show --name "githubActionsForResumeOIDC" --resource-group "$AZURE_RESOURCE_GROUP_FOR_IDENTITY" --query tenantId`

Script to show these three items:

```bash
echo AZURE_CLIENT_ID: $(az identity show --name "githubActionsForResumeOIDC" --resource-group "$AZURE_RESOURCE_GROUP_FOR_IDENTITY" --query clientId -o tsv)
echo AZURE_SUBSCRIPTION_ID: $(az account show --query id -o tsv)
echo AZURE_TENANT_ID: $(az identity show --name "githubActionsForResumeOIDC" --resource-group "$AZURE_RESOURCE_GROUP_FOR_IDENTITY" --query tenantId -o tsv)
```

### Defining the GitHub Action workflow

Rather than hardcode values like my Storage Account name and Endpoint Name, I'll make some GitHub Actions repository variables to store those.

* `AZURE_STORAGE_ACCOUNT_NAME`: name of the Azure Storage Account
* `AZURE_ENDPOINT_NAME`: name of the Azure Endpoint
* `AZURE_ENDPOINT_RESOURCE_GROUP`: resource group containing the Azure Endpoint
* `AZURE_ENDPOINT_CDN_PROFILE_NAME`: name of the Azure Front Door and CDN profile

I created `.github/workflows/main.yml` with steps to checkout the content, upload it to the Azure Storage Account, and flush the CDN so it knows there's new content available.

### Fix deprecated Node.js v16 usage

The Microsoft Learn example contained some steps which use node16 by default and this triggered a deprecation warning. Update them to use the latest version.
