# Wazuh cluster deployment for AWS

This pack provisions and configures a distributed Wazuh SIEM cluster in AWS.

## Default architecture

* Three Wazuh indexer EC2 instances across three Availability Zones
* One Wazuh manager master and two manager workers
* One Wazuh dashboard
* Internal Network Load Balancer listeners on TCP 1514 and 1515
* Internal Network Load Balancer listener on TCP 443 for the dashboard
* Encrypted gp3 EBS volumes
* AWS Systems Manager access
* KMS encryption keys
* S3 buckets for configuration backups and index snapshots
* CloudWatch alarms for instance and load balancer health
* Private subnets with outbound package access through a NAT gateway

TCP 1514 is routed to manager workers. TCP 1515 is routed to the manager master.

## Important points

1. Fixed EC2 instances are used rather than automatic replacement. Wazuh nodes maintain cluster state and should be recovered through controlled procedures.
2. Instances have no public IP addresses.
3. Run Ansible from a management host connected to the VPC, a corporate VPN, or another approved private administration path.
4. Review the current Wazuh release notes before deployment.
5. Replace example CIDRs, instance types, retention periods and names before production use.
6. Pin a tested Ubuntu AMI before production if immutable infrastructure is required.
7. The S3 snapshot playbook performs prerequisite checks but deliberately does not install an unverified indexer plugin.

## Prerequisites

```bash
terraform version
ansible --version
aws --version
jq --version
```

Install the required collection:

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

Confirm the deployment identity:

```bash
aws sts get-caller-identity
```

## Deployment

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
terraform init
terraform fmt -check
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

Generate the inventory:

```bash
cd ..
./scripts/render_inventory.sh
```

Run the installation:

```bash
cd ansible
ansible-playbook playbooks/00_preflight.yml
ansible-playbook playbooks/01_generate_installation_files.yml
ansible-playbook playbooks/02_install_indexers.yml
ansible-playbook playbooks/03_install_managers.yml
ansible-playbook playbooks/04_install_dashboard.yml
ansible-playbook playbooks/05_post_configuration.yml
ansible-playbook playbooks/06_health_checks.yml
```

Or run:

```bash
./scripts/deploy.sh
```

## Agent deployment

Linux:

```bash
sudo WAZUH_MANAGER="agent NLB DNS name" \
WAZUH_REGISTRATION_SERVER="agent NLB DNS name" \
bash scripts/install_agent_linux.sh
```

macOS:

```bash
sudo WAZUH_MANAGER="agent NLB DNS name" \
WAZUH_REGISTRATION_SERVER="agent NLB DNS name" \
WAZUH_PKG_URL="current official package URL" \
bash scripts/install_agent_macos.sh
```

Windows PowerShell:

```powershell
$env:WAZUH_MANAGER="agent NLB DNS name"
$env:WAZUH_REGISTRATION_SERVER="agent NLB DNS name"
$env:WAZUH_MSI_URL="current official MSI URL"
.\scripts\install_agent_windows.ps1
```

## Production acceptance checks

* No central component has a public IP address
* Dashboard access is limited to approved administration networks
* TCP 9200, 9300, 1516 and 55000 are restricted to required security groups
* Default Wazuh credentials are replaced
* Administrative multi factor authentication is enforced through the approved access path
* Positive and negative detection tests pass
* Worker, master and indexer failure tests pass
* Configuration restoration and snapshot restoration tests pass
* Critical asset and critical source coverage is 100 per cent
* Event ingestion is below two minutes
