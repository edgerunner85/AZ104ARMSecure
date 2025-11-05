This lab guides you through building a secure, end-to-end Azure environment aligned with AZ-104 objectives while emphasizing real-world cloud security tasks. 

Resources:
1. GitHub for Infrastructure-as-Code (ARM templates & Scripts)
2. Gitpod.io for an ephemeral CLI workspace
3. Azure resources
4. Local VMware nested virtualization node for offline testing.

Note: Every step is designed so any IT professional—or even a non-technical stakeholder—can follow along.

                      ┌──────────────┐
                      │  GitHub Repo │
                      │(ARM Templates│
                      │  & Scripts)  │
                      └──────┬───────┘
                             │  Git Clone & Gitpod.io Workspace
 ┌────────────┐          ┌───▼────────────┐          ┌───────────────┐
 │  Developer │ ──Web────►│   Gitpod.io   │──Azure CLI►│     Azure     │
 │  Machine   │          │(ARM, Azure CLI│          │ Subscription: │
 └────────────┘          │  & Python)     │          │ - VNets, NSGs  │
                         └───┬────────────┘          │ - VMs, KeyVault│
                             │ Git CLI & ARM         │ - Azure Policy │
                             ▼                       │ - Azure Monitor│
                     ┌─────────────────┐              └───────────────┘
                     │  Local VMware   │
                     │ Nested Hyper-V  │
                     │(Optional Offline│
                     │  VM Testing)    │
                     └─────────────────┘








Prerequisites
- Azure Subscription with Contributor rights

- GitHub Account

- Gitpod.io Accountor Railway Aaccount (free tier is sufficient)

- VMware Workstation/Fusion, VirtualBox, or Hyper-V (for optional local testing)



1. Bootstrap Your GitHub Repo
Create a new GitHub repository named az104-secure-lab.

Add directories:

/arm-templates

/scripts

/policy-definitions

Commit an initial ARM template stub in /arm-templates/main.json:

json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "resources": [],
  "outputs": {}
}
Push to GitHub and note the repo URL.

2. Launch Your Gitpod Workspace
In your browser, navigate to https://gitpod.io/#<your-github-repo-url>.

Gitpod will provision an Ubuntu container with:

Azure CLI

ARM CLI

PowerShell Core

Git

Confirm CLI tools:

bash
az --version
az bicep version
3. Secure Identity & Access
3.1 Create an Azure AD Group and Service Principal
bash
# Create a resource group for identity objects
az group create -n rg-identity-lab -l eastus

# Create Azure AD group
az ad group create \
  --display-name "CyberSecAdmins" \
  --mail-nickname "cybersecadmins"

# Create a Service Principal
az ad sp create-for-rbac \
  --name "sp-cybersec-lab" \
  --role Contributor \
  --scopes /subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/rg-identity-lab
3.2 Define and Assign Custom Role (Least Privilege)
Save /scripts/customRole.json:

json
{
  "Name": "VM Reader with Restart",
  "IsCustom": true,
  "Description": "Can read VMs and restart them",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/restart/action"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/rg-identity-lab"
  ]
}
Deploy and assign:

bash
az role definition create --role-definition ./scripts/customRole.json

az role assignment create \
  --role "VM Reader with Restart" \
  --assignee-object-id $(az ad sp show --id http://sp-cybersec-lab --query objectId -o tsv) \
  --scope /subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/rg-identity-lab
4. Deploy a Hardened Virtual Network and VM
4.1 ARM Template for VNet + NSG + Bastion
In /arm-templates/network.json, declare:

A VNet with two subnets:

subnet-app (for application VMs)

AzureBastionSubnet (for Azure Bastion Host)

A Network Security Group that allows only SSH (22), RDP (3389), and HTTPS (443) from your IP

Deploy with:

bash
az group create -n rg-secure-lab -l eastus

az deployment group create \
  --resource-group rg-secure-lab \
  --template-file arm-templates/network.json
4.2 Deploy Ubuntu VM with Managed Identity and Key Vault
Key Vault for storing VM secrets:

bash
az keyvault create \
  -n kv-lab-secrets \
  -g rg-secure-lab \
  -l eastus \
  --sku standard

az keyvault secret set \
  --vault-name kv-lab-secrets \
  --name "VmAdminPassword" \
  --value "<YourStrongP@ssw0rd>"
ARM Template /arm-templates/vm.json referencing the vault and enabling system-assigned identity.

Deploy:

bash
az deployment group create \
  -g rg-secure-lab \
  --template-file arm-templates/vm.json \
  --parameters adminUsername=azureuser \
              adminPasswordSecretName=VmAdminPassword \
              keyVaultName=kv-lab-secrets
5. Enforce Security Policies
5.1 Built-in Azure Policy Definitions
Assign the “Audit VMs that do not use managed disks” policy:

bash
az policy assignment create \
  --scope /subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/rg-secure-lab \
  --policy "e56962a8-e699-4d64-81b4-395fdcbd4d30" \
  --display-name "Audit Unmanaged Disks"
5.2 Custom Policy: Deny Public IPs on VMs
Place /policy-definitions/denyPublicIp.json:

json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {"field": "type", "equals": "Microsoft.Network/networkInterfaces"},
        {"field": "Microsoft.Network/networkInterfaces/ipConfigurations[*].publicIpAddress.id", "exists": true}
      ]
    },
    "then": {
      "effect": "Deny"
    }
  }
}
Assign it:

bash
az policy definition create \
  --name "deny-vm-publicip" \
  --display-name "Deny VM Public IP" \
  --rules policy-definitions/denyPublicIp.json \
  --mode Indexed

az policy assignment create \
  --policy "deny-vm-publicip" \
  --scope /subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/rg-secure-lab \
  --display-name "Deny VM Public IP"
6. Monitor & Alert with Azure Monitor
Enable Diagnostic Settings on your VM:

bash
az monitor diagnostic-settings create \
  --resource /subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/rg-secure-lab/providers/Microsoft.Compute/virtualMachines/securevm \
  --workspace /subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/rg-secure-lab/providers/Microsoft.OperationalInsights/workspaces/omslab \
  --logs '[{"category": "VMConnection", "enabled": true}]'
Create an Alert Rule for failed logins:

bash
az monitor metrics alert create \
  --name "FailedLoginAlert" \
  --resource-group rg-secure-lab \
  --scopes /subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/rg-secure-lab/providers/Microsoft.Compute/virtualMachines/securevm \
  --condition "count X where X >= 5" \
  --description "Alert on 5 failed SSH logins within 5 minutes"
7. Optional: Local VMware Nested VM Testing
Export your ARM templates to your local machine.

On VMware Workstation, spin up a nested Hyper-V host with Ubuntu VM.

Install Azure CLI and test the same ARM deployments locally.

Practice simulating network attacks via Kali Linux in the subnet-app network.

8. Cleanup
bash
az group delete --name rg-secure-lab --yes --no-wait
az group delete --name rg-identity-lab --yes --no-wait
Lab Takeaways
Hands-on practice with key AZ-104 domains: identity, networking, compute, storage, monitoring, and governance.

Real-world security configurations: NSGs, Bastion, Key Vault, Azure Policy, and Alerting.

Infrastructure-as-Code using ARM, GitHub, and Gitpod.

Bridged cloud-to-local testing with VMware nested virtual



                     

