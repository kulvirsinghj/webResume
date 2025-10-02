<div align="center">

# Cloud Resume Challenge Build


## The objective of this challenge was to build a static website, [digital resume](https://ksjandu.com), hosted in Azure and using cloud technologies.

</div>

As someone who has worked with cloud technologies in the past I felt it was time to build a simple project for myself and the [Cloud Resume
Challenge](https://cloudresumechallenge.dev/docs/the-challenge/azure/) was the perfect project to take on.

I had preemptively already purchased my [domain](https://ksjandu.com) through Cloudflare, which added an additional hurdle which I will cover later.

My first step was to create a simple eye catching site for my digital resume, using HTML, CSS and JavaScript. I don't claim to be a WebDev but I do know the basics however I found a few templates online and with the help of Copilot I was able to put together what you now currently see.

I already had an Azure account, which I had used for labs as I am also preparing for the AZ-104 exam. I used my existing subscription, where a new resource group was created, and the relevant resources for the project are stored.

First a General Purpose v2 (StorageV2) storage account was created, as I am located in Australia, Australia East region was the most logical choice and to save on costs Locally-redundant storage (LRS) was used for Data redundancy.

In the storage account, on the left side under Data Management, select Static website. We need to then enable this feature as it then tells the storage container to server the stored HTML, CSS and JS files directly to the browser. 

From here added index.html to the Index document name path, this is the name of the webpage that Azure Storage returns when a request is made to the root of the website or any subfolder. Once saved uploaded the files for the static site to the $web container which was created automatically after enabling "Static website".

Once files have been uploaded I was able to test if the site worked by navigating back to Static website option and browsing to the Primary endpoint link that had been created.

Now for getting the site accessible using the domain I had pre-purchased prior to starting this project. For this project Azure DNS was outlined to be used and using a subdomain would have resolved my issues however I wanted to use my FQDN for this project. I discovered when you purchase a domain from Cloudflare you're unable to change the NS records for the domain, however you're able to create a subdomain within Cloudflare and then point the NS records to the NS in the DNS Zone created in Azure.

As I wanted to use my domain, [ksjandu.com](https://ksjandu.com) this was not the route I went down. After some extensive research and trail and error. The solution I came to which then allowed for the site to have a SSL certificate for both [www.kjandu.com](https://ksjandu.com) and [ksjandu.com](https://ksjandu.com) was to use Azure Front Door and use that Endpoint hostname in the CNAME records in Cloudflare.

Now when visiting the Cloud Resume, you would have been greeted with a message stating that the site is not using HTTPS and its is not secure. We now needed to generate AFD (Azure Front Door) managed certificate. To configure HTTPS we needed to navigate back to Azure Front Door and go to Settings > Domain. Added both [www.kjandu.com](https://ksjandu.com) and [ksjandu.com](https://ksjandu.com). TXT records were also provided for validation which were then added to Cloudflare.

Finally it was time to setup the automation workflow using GitHub Action, this would allow for the static site to be updated in Azure when I pushed an updated to GitHub from, in my instance, VS Code.

A Service Principle account is needed to be created along with a secret to enter into GitHub. Using Azure CLI the following script below will create the Service Principle account and assign it Contributor role to your resource group where the resources for the static site is located, it will then also display an output in JSON format which is required in GitHub. 

```
az ad sp create-for-rbac - name "Service Principle Name" - role contributor - scopes /subscriptions/<Subscription ID>/resourceGroups/<Resource Group Name> - json-auth
```

Lastly we needed to configure the CI pipeline for deploying the static website to Azure and update it live. Below is the YAML file that was used.


```
name: Blob storage website CI

on:
  push:
    branches: [ main ] # Trigger this workflow when code is pushed to the main branch

jobs:
  build:
    runs-on: ubuntu-latest # Run job on GitHub-hosted Ubuntu VM
    steps:
    # Step 1: Checkout repository code so we can deploy it
    - uses: actions/checkout@v3
     # Step 2: Login to Azure using service principal credentials stored in GitHub Secrets
    - uses: azure/login@v1
      with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
    # Step 3: Upload all files in the repo to Azure Blob Storage's $web container
    - name: Upload to blob storage
      uses: azure/CLI@v1
      with:
        inlineScript: |
            az storage blob upload-batch \
            --account-name <Account Name> \ # Name of the Azure Storage account
            --auth-mode key \               # Authenticate with storage account key
            -d '$web' \                     # Destination container
            -s . \                          # Source directory = repo root
            --overwrite                     # Overwrite existing files
           
    # Step 4: Logout of Azure (always runs, even if job fails)
    - name: logout
      run: |
            az logout
      if: always()
```