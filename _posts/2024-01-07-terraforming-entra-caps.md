---
title: "Terraforming Your Existing Entra ID Conditional Access Policies"
date: 2025-07-01T07:32:30+02:00
#last_modified_at:
categories: [PowerShell, Terraform]
tags:
  - Terraform
  - PowerShell
  - Entra ID
  - Conditional Access
---

This guide demonstrates how to import existing Microsoft Entra ID Conditional Access Policies into Terraform using PowerShell, enabling Infrastructure as Code without any manual policy recreation.

## Pre-requisites
1. You have Terraform [installed](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli#install-terraform).
2. [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) to run Terraform interactively.
3. Microsoft Graph PowerShell SDK or at least [Microsoft.Graph.Authentication](https://www.powershellgallery.com/packages/Microsoft.Graph.Authentication).
4. Entra ID Permissions ('Policy.Read.All')- global reader is probably your best bet.

For your convenience, all the scripts and required terraform files are available in the [blog-code-snippets](https://github.com/alflokken/blog-code-snippets/tree/main/terraform-ca-import) repository.

# Steps to import Conditional Access Policies (CAPs)

## 1. Setup the environment
In the working directory, configure the `providers.tf` with your [tenant_id](https://learn.microsoft.com/en-us/sharepoint/find-your-office-365-tenant-id).

```hcl
terraform {
  required_version = ">= 1.11.0"
  required_providers {
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 3.3.0"
    }
  }
}
provider "azuread" {
  tenant_id = "f9373f4a-a17e-4478-b0c8-1507ef1401cf"
}
```

Run `terraform init` to initialize the Terraform directory and download the required providers:

![terraform init]({{site.baseurl}}/assets/img/2024-01-07-terraforming/init.png){: .normal }

## 2. Generate terraform import config
Run the `.\1_generate-imports.ps1` script to generate `imports.tf`.

```powershell
# Retrieve tenant_id from providers.tf
$tenantId = ((Get-Content .\providers.tf | Where-Object { $_ -match "tenant_id" }) -split '"')[1]

# Ensure Graph Connection with required scope
$mgContext = Get-MgContext
if (-not $mgContext -or
    ($mgContext.TenantId -ne $tenantId) -or
    ("Policy.Read.All" | Where-Object { $_ -notin $mgContext.scopes }).Count
) { Connect-MgGraph -TenantId $tenantId -Scopes "Policy.Read.All" -NoWelcome -ErrorAction Stop -Debug:$false }

$graphData = @()
"namedLocations","policies" | ForEach-Object {
    $graphData += Invoke-GraphRequest -Uri "v1.0/identity/conditionalAccess/$_/`?select=id,displayName" -OutputType PSObject
}

$content = @()
foreach ( $query in $graphData ) {

  if ( $query.'@odata.context' -match "namedLocations" ) { 
    $identifierType = "namedLocations"
    $destinationType = "named_location"
  }
  else { 
    $identifierType = "policies"
    $destinationType = "conditional_access_policy"
  }

  # terraform import code block formatting
  foreach ( $object in $query.value ) {
    $content += "import {"
    $content += "    id = `"identity/conditionalAccess/$identifierType/$($object.id)`""
    $content += "    to = azuread_$destinationType.$($object.displayName -replace '\s', '_' -replace '\W')"
    $content += "}"
  }
}
$content | Out-File "imports.tf" -Encoding utf8 -Force
```

The resulting `imports.tf` will contain the necessary Terraform import statements for our CA and named locations: 
![imports.tf sample]({{site.baseurl}}/assets/img/2024-01-07-terraforming/imports.png){: .normal }

## 3. Import the configuration
Run `terraform plan -generate-config-out="generated.tf"` to generate terraform configuration in `generated.tf`.

> Expect errors like below, config generation is still [experimental](https://developer.hashicorp.com/terraform/language/import/generating-configuration). We will clean these up in the next step!
{: .prompt-warning }
![import errors]({{site.baseurl}}/assets/img/2024-01-07-terraforming/errors.png){: .normal }

The resulting file will look similar to this:
![generated.tf]({{site.baseurl}}/assets/img/2024-01-07-terraforming/generated.png){: .normal }

## 4. Clean Up the Config
Run the `.\3_clean_files.ps1` script to clean the generated config files and split them into multiple files, this will get rid of most of the errors and warnings in the previous step.

Named locations will be saved to `_named_locations.tf`.

```powershell
# Clean up the generated terraform files and split them into multiple files prefixed by underscore.
$path = "generated.tf"

# removes empty blocks and null values (including sign_in_frequency = 0 (bug in the terraform provider))
$fileContent = Get-Content $path -Encoding UTF8
$fileContent | Where-Object { $_ -notmatch "\[\]$|=\snull$|sign_in_frequency\s+=\s0$" } | Out-File $path -Encoding utf8 -Force

# split the file into multiple files
$fileContent = Get-Content $path -Raw
$resources = $fileContent -split "# __generated__ by Terraform.*\r?\n" | Where-Object { $_ -match "^resource" }

# named locations go in one file
$namedLocationsContent = @()
foreach ( $res in $resources ) {
    # extract the resource type and name from the first line
    $metadata = ($res -split "\r?\n" )[0] -replace "`"" -split "\s"
    $type = $metadata[1]
    $name = $metadata[2]

    if ( $type -match "azuread_conditional_access_policy" ) {  $res | Out-File "$name.tf" -Encoding utf8 -Force }
    elseif ( $type -match "azuread_named_location" ) { $namedLocationsContent += $res }
    else { Write-Warning "Unknown resource type: $type" }
}
$namedLocationsContent | Out-File "_named_locations.tf" -Encoding utf8 -Force

# Fix the formatting
terraform fmt | Out-Null

# remove files that are no longer needed
remove-item $path -Force
remove-item imports.tf -Force
```

The resulting files will look similar to this:
![clean.tf]({{site.baseurl}}/assets/img/2024-01-07-terraforming/clean.png){: .normal }

## 5. Moment of truth
Run `terraform validate` to validate the configuration. If everything is correct, you should see a message like this:
![clean.tf]({{site.baseurl}}/assets/img/2024-01-07-terraforming/valid.png){: .normal }

If you still encounter errors, you may need to resolve these manually, the 'AzureAd' Terraform provider is under development and new Conditional Access Policy features [often lag behind](https://github.com/hashicorp/terraform-provider-azuread/issues/1657). 

## Known Issues
* Terraform does currently not support 'Authentication context' as a condition, [only user actions or resources](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/conditional_access_policy#applications-1) is supported as of now. 