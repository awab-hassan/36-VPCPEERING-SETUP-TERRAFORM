# Cross-Region VPC Peering (Tokyo ↔ Sydney)

Terraform module that establishes a full cross-region VPC peering link between **Tokyo (ap-northeast-1)** and **Sydney (ap-southeast-2)** — creating the peering connection, auto-accepting it in the remote region, and injecting the return routes into every route table on both sides so instances in either VPC can reach each other over private AWS backbone.

## Highlights

- **Dual-region provider aliases** — one Terraform apply orchestrates two AWS regions simultaneously via `aws.tokyo` and `aws.sydney` aliases.
- **Auto-accept on the remote side** — the requester (Tokyo) sets `auto_accept = false`, and an `aws_vpc_peering_connection_accepter` in Sydney takes care of the acceptance — so the whole handshake is one `terraform apply`.
- **Route-table fan-out via `data` + `count`** — `data.aws_route_tables` discovers every route table in each VPC, and `count = length(data.ids)` creates a return route in each one. No hand-maintained list.
- **References existing VPCs** — the module assumes the two VPCs already exist and reads them via `data.aws_vpc`, so it cleanly slots into an environment you don't fully own.
- **Pairs with an in-region Lambda** — `lambda-sydney/` holds the Node.js handler that runs in the peered Sydney VPC, reachable from Tokyo once peering is up.

## Architecture
![Architecture Diagram](./architecture.png)

## Tech stack

- **Terraform** >= 1.x, AWS provider
- **AWS services:** VPC, VPC Peering, Route Tables, Lambda (in the Sydney VPC)
- **Regions:** `ap-northeast-1` (Tokyo), `ap-southeast-2` (Sydney)

## Repository layout

```
VPC-PEERING/
├── README.md
├── .gitignore
├── providers.tf            # aws.tokyo + aws.sydney aliases
├── data.tf                 # data.aws_vpc + data.aws_route_tables in both regions
├── main.tf                 # peering connection, accepter, routes
├── output.tf               # peering connection ID output
```

## How it works

1. The `aws.tokyo` provider creates the peering request (`aws_vpc_peering_connection`) pointing at the Sydney VPC ID with `peer_region = "ap-southeast-2"` and `auto_accept = false`.
2. The `aws.sydney` provider accepts it immediately (`aws_vpc_peering_connection_accepter` with `auto_accept = true`).
3. `data.aws_route_tables` on each side returns the list of route table IDs attached to the two VPCs.
4. `aws_route` with `count = length(...)` adds the return route to each route table — so every subnet gets the cross-region path with no manual wiring.
5. A Node.js Lambda (see `lambda-sydney/`) can then sit in the Sydney VPC and be reached from Tokyo services over the peering.

## Prerequisites

- Terraform >= 1.x
- AWS CLI configured with credentials permitted in both regions for `ec2:CreateVpcPeeringConnection`, `ec2:AcceptVpcPeeringConnection`, `ec2:CreateRoute`
- Existing VPCs in each region — update the IDs in `data.tf` (`data.aws_vpc.tokyo_vpc.id` and `.sydney_vpc.id`)
- Non-overlapping CIDR blocks in the two VPCs

## Deployment

```bash
terraform init
terraform plan
terraform apply
```

## Teardown

```bash
terraform destroy
```

## Notes

- The CIDRs `10.50.0.0/16` (Tokyo) and `10.0.0.0/16` (Sydney) are hard-coded in `main.tf` — parameterise via variables if re-using.
- Demonstrates: multi-region Terraform (provider aliases), data-driven route fan-out, cross-region private networking without Transit Gateway.
