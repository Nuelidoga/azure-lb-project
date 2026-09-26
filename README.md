# Azure Load Balancer Demo — VNet, ARM, CI/CD, Monitoring

A simple 2-VM web app behind an Azure Load Balancer, provisioned entirely
with ARM templates and deployed via GitHub Actions, with Azure Monitor
alerting on health probe failures.

## Architecture

Internet → Public IP → Standard Load Balancer → 2x Ubuntu VMs (nginx, no
public IPs, single subnet) → NSG allows inbound port 80.

Monitoring: Log Analytics workspace + metric alert on load balancer
DipAvailability, routed to an email action group.

## AWS → Azure mapping

| AWS concept | Azure equivalent |
|---|---|
| VPC + subnets + security groups | Virtual Network + subnet + NSG |
| EC2 instances | 2x Ubuntu VMs (cloud-init installs nginx) |
| Elastic Load Balancer | Azure Standard Load Balancer |
| Terraform | ARM template (deployed via Azure CLI) |
| CodePipeline / GitHub Actions | GitHub Actions + azure/login |
| CloudWatch | Azure Monitor (Log Analytics, metric alerts, action groups) |

## Deploying

1. `az login`
2. `az group create -n rg-lbdemo -l uksouth`
3. `az deployment group create -g rg-lbdemo -f infra/azuredeploy.json -p sshPublicKey="<your key>" alertEmail="<your email>"`

Or push to `main` — the GitHub Actions workflow in
`.github/workflows/deploy.yml` validates, runs a what-if, and deploys
automatically.

## Incident: probe port misconfiguration

See [`docs/incident-report.md`](docs/incident-report.md) for a full
write-up of an intentional outage: a load balancer probe port mismatch
that took the site down for ~21 minutes, how it was detected via Azure
Monitor, and how it was diagnosed and fixed through the pipeline.

## Screenshots

(add your screenshots here — working site alternating between VMs, green
pipeline run, the metric V-dip, the alert, the recovered metric)

## Cleanup

`az group delete -n rg-lbdemo --yes --no-wait`
