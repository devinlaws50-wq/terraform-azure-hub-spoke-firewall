# Azure Hub-and-Spoke Firewall Architecture with Terraform

[![Terraform Validate](https://github.com/devinlaws50-wq/terraform-azure-hub-spoke-firewall/actions/workflows/terraform-validate.yml/badge.svg)](https://github.com/devinlaws50-wq/terraform-azure-hub-spoke-firewall/actions/workflows/terraform-validate.yml)

![Platform](https://img.shields.io/badge/Platform-Microsoft%20Azure-0078D4?style=for-the-badge)
![IaC](https://img.shields.io/badge/IaC-Terraform-844FBA?style=for-the-badge)
![Network](https://img.shields.io/badge/Network-Hub%20and%20Spoke-0A66C2?style=for-the-badge)
![Security](https://img.shields.io/badge/Security-Azure%20Firewall-FF6F00?style=for-the-badge)
![Access](https://img.shields.io/badge/Access-Azure%20Bastion-4CAF50?style=for-the-badge)
![Monitoring](https://img.shields.io/badge/Monitoring-Log%20Analytics-8E44AD?style=for-the-badge)

## Project Overview

This project implements a secure Azure hub-and-spoke network architecture using Terraform.

Azure Firewall is centralized in the hub virtual network and acts as the primary egress security control for spoke workloads. User-defined routes force internet-bound traffic from the spoke networks through the firewall, while Azure Bastion provides administrative access to private virtual machines without requiring public IP addresses.

Firewall diagnostics are forwarded to Log Analytics, allowing both permitted and denied traffic to be validated through KQL queries.

The project demonstrates how networking, routing, security enforcement, private administrative access, and centralized monitoring work together in a segmented Azure environment.

---

## Business and Security Problem

As cloud environments grow, independently managing routing, internet access, and security controls in every workload network can create inconsistent policies and unnecessary operational overhead.

A centralized hub-and-spoke design helps address this by placing shared network and security services in a central hub while keeping workloads separated across spoke networks.

This architecture was designed to demonstrate:

- Centralized network security enforcement
- Controlled internet egress
- Segmentation between workloads
- Private administrative access
- Centralized traffic logging and monitoring
- Repeatable infrastructure deployment using Terraform

---

## Architecture

The environment uses one hub virtual network and two spoke virtual networks.

The hub hosts Azure Firewall and Azure Bastion, while workload virtual machines reside in the spoke networks. VNet peering provides connectivity between the hub and spokes.

User-defined routes on the spoke subnets send default outbound traffic to the Azure Firewall private IP, creating a centralized inspection point for internet-bound traffic.

```mermaid
flowchart LR
    Internet((Internet))
    LA[(Log Analytics)]
    Bastion[Azure Bastion]

    subgraph Hub["Hub VNet: vnet-hub-devin"]
        FW["Azure Firewall\nfw-hub-devin\nPrivate IP: 10.0.1.4"]
    end

    subgraph Spoke1["Spoke 1 VNet: vnet-spoke1-devin"]
        VM1["vm-spoke1-01\n10.1.1.4"]
    end

    subgraph Spoke2["Spoke 2 VNet: vnet-spoke2-devin"]
        VM2["vm-spoke2-01\n10.2.1.4"]
    end

    VM1 -->|0.0.0.0/0 via UDR| FW
    VM2 -->|0.0.0.0/0 via UDR| FW
    FW --> Internet

    Bastion --> VM1
    FW --> LA

    Hub <-->|VNet Peering| Spoke1
    Hub <-->|VNet Peering| Spoke2

Technologies Used
    | Technology             | Purpose                                       |
| ---------------------- | --------------------------------------------- |
| Terraform              | Infrastructure as Code deployment             |
| Microsoft Azure        | Cloud platform                                |
| Azure Virtual Network  | Hub and spoke network segmentation            |
| VNet Peering           | Connectivity between hub and spoke VNets      |
| Azure Firewall         | Centralized traffic filtering                 |
| Azure Firewall Policy  | Application and network rule management       |
| User-Defined Routes    | Forced egress through Azure Firewall          |
| Azure Bastion          | Private administrative access to VMs          |
| Linux Virtual Machines | Spoke workloads used for validation           |
| Log Analytics          | Centralized firewall log collection           |
| Azure Monitor          | Monitoring and diagnostic integration         |
| Network Watcher        | Connectivity validation                       |
| GitHub Actions         | Automated Terraform formatting and validation |

Security Controls
Centralized Egress Filtering

Default routes from the spoke subnets direct internet-bound traffic through Azure Firewall rather than allowing workloads to communicate directly with the internet.

Private Workload Access

The Linux virtual machines use private IP addresses and do not require directly exposed public IP addresses for administrative access.

Azure Bastion provides browser-based SSH connectivity to the private workload VM.

SSH Key Authentication

Linux VM authentication uses SSH public keys rather than password-based authentication.

The SSH public key is supplied through a Terraform variable so the configuration does not depend on a specific local workstation path.

Network Segmentation

Separate spoke VNets provide logical separation between workload environments while the hub centralizes shared infrastructure.

Application-Level Filtering

Azure Firewall application rules allow approved FQDN destinations while unmatched traffic is denied.

Centralized Logging

Firewall diagnostic logs are sent to Log Analytics for query, investigation, and validation.

Infrastructure Validation

GitHub Actions automatically performs:
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
on pushes to main and pull requests.

Deployment

The environment was deployed into:
rg-wk17-hub-spoke-devin
in the East US Azure region.

Terraform provisions the major infrastructure components, including:

Hub virtual network
Two spoke virtual networks
VNet peering
Azure Firewall
Azure Firewall Policy
Route tables
Linux workload VMs
Azure Bastion
Log Analytics integration
Diagnostic settings

Terraform Workflow
cd terraform

terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
Local environment-specific values should be supplied through a local terraform.tfvars file based on:
terraform.tfvars.example
Terraform state and local variable files are intentionally excluded from source control.

Validation

Deployment validation focused on proving that traffic followed the intended path and that firewall policy decisions were enforced.

Resource Group Validation

The Azure resource group contained the infrastructure components required for the hub-and-spoke environment.

Azure Firewall

Azure Firewall was deployed inside the hub VNet as the centralized egress security control.

Firewall Policy

The firewall was associated with the fwp-hub-devin policy used to manage application and network filtering rules.

VNet Peering

The hub VNet was peered with both spoke VNets to enable connectivity to centralized hub services.

Forced Egress Routing

A user-defined route sends:

0.0.0.0/0

from the spoke subnet to the Azure Firewall private IP:

10.0.1.4

This forces internet-bound traffic through the firewall.

Private Workload Addressing

The workload VMs use private addressing:

vm-spoke1-01 → 10.1.1.4
vm-spoke2-01 → 10.2.1.4

Diagnostic Logging

Azure Firewall diagnostic settings forward firewall activity into Log Analytics for centralized visibility.

Firewall Policy Testing

Testing was performed from vm-spoke1-01 through Azure Bastion.

Requests were generated to:

www.microsoft.com
www.bing.com
example.com

Microsoft and Bing were allowed, while example.com remained denied.

This validated selective application-rule enforcement rather than unrestricted outbound access.

Application Rule

The app-allow-web application rule collection used:

Priority: 100
Action: Allow
Source: 10.1.1.4

with approved destination FQDNs.

Network Watcher Validation

Network Watcher Connection Monitor was configured as an additional connectivity-validation mechanism.

Log Analysis

Azure Firewall logs were queried through Log Analytics.

The following KQL query was used to isolate application-rule activity originating from the spoke workload:

AzureDiagnostics
| where Category == "AzureFirewallApplicationRule"
| where TimeGenerated > ago(30m)
| where msg_s has "10.1.1.4"
| project TimeGenerated, msg_s
| order by TimeGenerated desc

The results showed:

Allow decisions for www.microsoft.com
Allow decisions for www.bing.com
Deny decisions for example.com

This confirmed that firewall policy decisions were both enforced and recorded.

Challenges and Troubleshooting
Traffic Routing

Deploying Azure Firewall alone does not automatically force spoke traffic through it.

User-defined routes had to be associated with the spoke subnets so their default route pointed to the firewall private IP.

Firewall Rule Validation

Successful connectivity by itself was not enough to prove the firewall policy was functioning correctly.

Both allowed and denied destinations were tested so that rule behavior could be validated from the workload and again through firewall logs.

CI Portability

The original Terraform configuration referenced a workstation-specific SSH public-key path:

~/.ssh/id_rsa.pub

GitHub Actions correctly identified that this dependency would not exist on another runner.

The configuration was refactored to accept the SSH public key through:

var.ssh_public_key

making the Terraform configuration portable across environments.

Security Considerations

This project follows several security-focused design practices:

Workload VMs do not require directly exposed public IPs for administration.
Administrative access is provided through Azure Bastion.
Password authentication is disabled on the Linux VMs.
SSH public-key authentication is used instead.
Internet-bound traffic is centrally routed through Azure Firewall.
Application rules restrict permitted outbound destinations.
Firewall logs are centrally collected for analysis.
Terraform state and local .tfvars files are excluded from Git.
GitHub Actions operates with read-only repository permissions for validation.
CI performs validation without requiring Azure credentials or deploying infrastructure.

For a production environment, additional improvements could include:

Remote Terraform state with locking
Azure Key Vault integration
Private DNS architecture
Azure Firewall Premium capabilities
Additional NSG controls
Centralized identity integration
Multi-environment Terraform modules
Policy-as-code enforcement
Cost Considerations

Azure Firewall and Azure Bastion are billable resources and can create meaningful costs even in a lab environment.

For temporary testing environments:

Deploy only when actively testing.
Monitor Azure Cost Management.
Destroy unused resources promptly.
Avoid leaving Azure Firewall or Bastion running unnecessarily.
Cleanup

When the environment is no longer required:

cd terraform
terraform destroy

Review the destruction plan before confirming.

Because Terraform state is intentionally not committed to GitHub, destruction should be performed from the environment containing the corresponding local or remote state.

Key Outcomes

This project demonstrated the ability to:

Design a segmented Azure hub-and-spoke architecture
Deploy Azure infrastructure using Terraform
Centralize outbound filtering through Azure Firewall
Configure forced routing using UDRs
Secure administrative access with Azure Bastion
Implement SSH key authentication
Apply application-level firewall rules
Forward firewall diagnostics to Log Analytics
Analyze allowed and denied traffic with KQL
Validate network behavior through multiple tools
Build portable Terraform configurations
Implement automated Terraform CI validation with GitHub Actions
Skills Demonstrated
Azure Networking
Azure Firewall
Hub-and-Spoke Architecture
Terraform
Infrastructure as Code
VNet Peering
User-Defined Routes
Azure Bastion
Linux Administration
SSH Security
Log Analytics
Azure Monitor
KQL
Network Watcher
Network Segmentation
Firewall Policy Management
Cloud Security
Git
GitHub Actions
CI/CD
Repository Structure
.
├── .github/
│   └── workflows/
│       └── terraform-validate.yml
├── diagrams/
│   └── wk17-diagram.png
├── screenshots/
│   ├── wk17-build-01-resource-group.png
│   ├── wk17-build-03-firewall-overview.png
│   ├── wk17-build-04-vm-private-ips.png
│   ├── wk17-hub-spoke-01-firewall-policy.png
│   ├── wk17-hub-spoke-02-vnet-peering.png
│   ├── wk17-hub-spoke-03-udr-route-table.png
│   ├── wk17-hub-spoke-04-flow-logs-la.png
│   ├── wk17-hub-spoke-05-bastion-curl.png
│   ├── wk17-hub-spoke-06-fw-app-rule.png
│   ├── wk17-hub-spoke-06-network-watcher-capture.png
│   └── wk17-hub-spoke-07-fw-logs-allow-deny.png
├── terraform/
│   ├── .terraform.lock.hcl
│   ├── main.tf
│   ├── outputs.tf
│   ├── provider.tf
│   ├── terraform.tfvars.example
│   ├── variables.tf
│   └── versions.tf
├── .gitignore
└── README.md