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

The general process follows [Microsoft Learn: Use GitHub Actions workflow to deploy your static website in Azure Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-static-site-github-actions?tabs=userlevel)

### Setting up access into Azure

The example given in the above is:

```text
az ad sp create-for-rbac --name "myML" --role contributor --scopes /subscriptions/<subscription-id>/resourceGroups/<group-name> --json-auth
```

Which allows the GitHub action to make changes to any resource in the given resource group, which is more than necessary to upload content to a storage account and purge a CDN endpoint. I'll limit the scopes of the Service Principal I create to something more appropriately limited. To make the command copy-paste-able, I'll set some configuration via environment variables which contain the Azure Resource ID for each of the the above resources:

```bash
export AZURE_STORAGE_ACCOUNT_ID=resource-id-for-the-web-content-storage-account
export AZURE_ENDPOINT_ID=resource-id-for-the-azure-endpoint
```

Create the Service Principal using my Azure Owner account with the scope of the storage account and endpoint:

```bash
az ad sp create-for-rbac \
  --name "githubActionsForResume" \
  --role contributor \
  --scopes "$AZURE_STORAGE_ACCOUNT_ID" "$AZURE_ENDPOINT_ID" \
  --json-auth
```

The output was saved in my secure password storage.

### Providing the Service Principal credential to GitHub

I'm using the GitHub web interface for simplicity, but this could be done more consistently and reproducibly using the GitHub CLI via `gh secret set`. The credential will be scoped to this repository only since this is the only place which needs access to those Azure resources.

I named this credential `AZURE_SP_GITHUBACTIONSFORRESUME` to show that it's associated with an Azure Service Principal and the Service Principal's name.

### Defining the GitHub Action workflow

Rather than hardcode values like my Storage Account name and Endpoint Name, I'll make some GitHub Actions repository variables to store those.

* `AZURE_STORAGE_ACCOUNT_NAME`: name of the Azure Storage Account
* `AZURE_ENDPOINT_NAME`: name of the Azure Endpoint
* `AZURE_ENDPOINT_RESOURCE_GROUP`: resource group containing the Azure Endpoint
* `AZURE_ENDPOINT_CDN_PROFILE_NAME`: name of the Azure Front Door and CDN profile

I created `.github/workflows/main.yml` with steps to checkout the content, upload it to the Azure Storage Account, and flush the CDN so it knows there's new content available.
