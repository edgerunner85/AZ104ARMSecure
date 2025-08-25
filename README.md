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

