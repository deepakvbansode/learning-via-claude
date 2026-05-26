# OpenStack — Consuming It & Deploying a 3-Tier Application

## The Problem It Solves

Without OpenStack, running a private cloud means managing bare metal directly — no self-service VM provisioning, no software-defined networking, no elastic storage. OpenStack gives your org AWS-like infrastructure primitives (compute, network, storage, LB) on hardware they control, with a tenant model that lets teams operate independently without stepping on each other.

---

## Core Concepts

### Service Map + Project Model

**Why:** OpenStack is a collection of named services, not one product. You need the code names to read logs, configs, and docs. The Project is your isolation boundary — everything you create lives inside one.

**Mental model:** A Project is like a Kubernetes namespace more than an AWS account — lightweight, quota-bounded, and a single user can belong to many. The underlying control plane (Nova, Neutron, Cinder) is shared across all projects.

**Example:**

```
AWS                  OpenStack Service     Code Name
─────────────────────────────────────────────────────
Account/Namespace  → Project / Tenant    → Keystone
EC2 Instances      → Instances           → Nova
VPC + Subnets      → Virtual Network     → Neutron
Elastic IP         → Floating IP         → Neutron
EBS Volumes        → Volumes             → Cinder
AMI                → Images              → Glance
S3                 → Object Storage      → Swift
ALB/NLB            → Load Balancer       → Octavia
CloudFormation     → Stacks              → Heat
```

---

### Keystone: Auth & Credentials

**Why:** Every API call to any OpenStack service must prove identity and project scope. Keystone centralises this — it issues short-lived tokens that all other services validate.

**Mental model:** Keystone = IAM + STS combined. You send credentials + project name, get back a scoped token (like an STS session token). The token is what actually talks to Nova/Neutron/Cinder — your password never leaves Keystone.

**Example:**

```yaml
# ~/.config/openstack/clouds.yaml — equivalent to ~/.aws/credentials
clouds:
  my-org-prod:
    auth:
      auth_url: https://keystone.my-org.com:5000/v3
      username: deepak
      password: secret
      project_name: team-backend
      user_domain_name: Default
      project_domain_name: Default
    region_name: RegionOne
```

```bash
export OS_CLOUD=my-org-prod
openstack server list

# Terraform reads the same file
# provider "openstack" { cloud = "my-org-prod" }
```

---

### Neutron: Virtual Networking

**Why:** Neutron provides software-defined networking per project — isolated L2 networks, routable subnets, and controlled external access via floating IPs. Unlike AWS VPC where routing is implicit, in OpenStack you assemble each piece explicitly.

**Mental model:** Think of it as AWS VPC where the Internet Gateway and Route Table association are separate manual steps — you create the router, attach it to the external network, and wire your subnet to it yourself.

**Example:**

```bash
openstack network create app-network
openstack subnet create app-subnet \
  --network app-network \
  --subnet-range 192.168.10.0/24 \
  --dns-nameserver 8.8.8.8

openstack router create app-router
openstack router set app-router --external-gateway external-net  # admin-created network
openstack router add subnet app-router app-subnet

# Floating IP for inbound external access (attach to LB, not instances)
openstack floating ip create external-net
openstack floating ip set --port <lb-vip-port-id> <floating-ip>
```

```
AWS                          OpenStack Neutron
────────────────────────────────────────────────────────
VPC                      →  Network
Subnet                   →  Subnet
Internet Gateway         →  External Network (admin-managed)
Route Table + IGW route  →  Router (you create and wire)
Elastic IP               →  Floating IP
Security Group           →  Security Group (stateful, same as AWS)
```

---

### Nova: Compute

**Why:** Nova manages the lifecycle of virtual machines — launch, stop, resize, terminate. It's the EC2 of OpenStack.

**Mental model:** Identical to EC2: pick an image (AMI), a flavor (instance type), a network, and a keypair. The one non-obvious difference is the root disk model — flavors with `disk > 0` give you ephemeral local storage; flavors with `disk = 0` require you to boot from a Cinder volume.

**Example:**

```bash
# Stateless app server — ephemeral root disk is fine
openstack server create \
  --flavor m1.large \
  --image ubuntu-22.04 \
  --network app-network \
  --security-group sg-app \
  --key-name my-key \
  --availability-zone az1 \
  app-server-1

# DB server — boot from Cinder volume so root disk survives termination
openstack server create \
  --flavor m1.large \
  --image ubuntu-22.04 \
  --boot-from-volume 50 \
  --network app-network \
  --security-group sg-db \
  --key-name my-key \
  --availability-zone az1 \
  db-server
```

```
Root disk strategy:
  Stateless (web, app tier)  → ephemeral (flavor disk > 0) — simpler, no quota cost
  Stateful (DB tier)         → boot-from-volume — survives termination, rebuildable
```

---

### Cinder + Swift: Storage

**Why:** Two different storage problems require two different tools. Cinder is a block device you mount into a VM (persistent, VM-local). Swift is object storage you access over HTTP from anywhere.

**Mental model:** Cinder = EBS (attach to one instance, block device, survives termination). Swift = S3 (HTTP API, no VM needed, pipeline-accessible from anywhere).

**Example:**

```bash
# Cinder — persistent block volume for DB data directory
openstack volume create --size 100 --availability-zone az1 db-data-vol
openstack server add volume db-server db-data-vol
# Inside instance: mkfs.ext4 /dev/vdb && mount /dev/vdb /var/lib/postgresql

# Swift — store CI/CD build artifacts
openstack object create my-bucket build-artifact-v1.tar.gz
```

```
Need                                    Use
──────────────────────────────────────────────────────
Persistent disk for DB data          → Cinder volume
Root disk surviving termination      → Cinder boot-from-volume
Build artifacts in CI/CD pipeline    → Swift
Database backups                     → Swift
Static assets served to users        → Swift
Shared filesystem across instances   → Manila/NFS (not Cinder or Swift)
```

---

### Octavia + Availability Zones

**Why:** Octavia distributes traffic across your instances and health-checks them. AZs let you spread instances across isolated failure domains. Together they determine your resilience posture.

**Mental model:** Octavia = AWS ALB. The object model is LB → Listener → Pool → Members. One floating IP attaches to the LB VIP — backend instances stay on private IPs only. AZs in OpenStack map directly to AWS AZs: physical isolation, same network reachability.

**Example:**

```bash
openstack loadbalancer create --name app-lb --vip-subnet-id app-subnet
openstack loadbalancer listener create \
  --name app-listener --protocol HTTP --protocol-port 80 app-lb
openstack loadbalancer pool create \
  --name app-pool --lb-algorithm ROUND_ROBIN \
  --listener app-listener --protocol HTTP
openstack loadbalancer member create \
  --address 192.168.10.11 --protocol-port 8080 app-pool
openstack loadbalancer member create \
  --address 192.168.10.12 --protocol-port 8080 app-pool
openstack loadbalancer healthmonitor create \
  --type HTTP --url-path /health --delay 10 --timeout 5 --max-retries 3 app-pool

# Attach floating IP to LB VIP port (not to instances)
openstack floating ip set --port <lb-vip-port-id> <floating-ip>
```

```
# Spread app instances across AZs
openstack server create --availability-zone az1 app-server-1
openstack server create --availability-zone az2 app-server-2
# DB stays in one AZ — Cinder volume must match
```

---

### 3-Tier Deployment Pattern

**Why:** Wiring compute, network, storage, and LB together in the right order is non-trivial — dependencies flow downward and provisioning order matters.

**Mental model:** Same pattern as K8s (Ingress → Service → Pod) or AWS (ALB → ASG → RDS), just different primitives. Security groups reference each other by ID (not CIDR) between tiers — same as AWS.

**Example:**

```
Architecture:
  Internet → [Floating IP] → [Octavia LB] → [App Server az1, App Server az2]
                                                         ↓
                                               [DB Server az1] + [Cinder Volume az1]

Terraform provisioning order:
  1. network + subnet + router          (Neutron foundation)
  2. security groups                    (referenced at instance launch)
  3. keypair
  4. db instance (az1) + cinder volume (az1)
  5. app instances (az1, az2)
  6. load balancer + listener + pool + members + health monitor
  7. floating ip → attach to LB VIP

Security group design:
  sg-lb:  inbound 0.0.0.0/0:80,443 | outbound → sg-app:8080
  sg-app: inbound sg-lb:8080, sg-bastion:22  | outbound → sg-db:5432
  sg-db:  inbound sg-app:5432, sg-bastion:22
```

```hcl
# Guard against AZ mismatch in Terraform
variable "db_az" { default = "az1" }

resource "openstack_compute_instance_v2" "db" {
  availability_zone = var.db_az
}

resource "openstack_blockstorage_volume_v3" "db_data" {
  availability_zone = var.db_az   # same variable — mismatch is impossible
  size              = 100
}
```

```bash
# Zero-downtime rolling deploy
openstack loadbalancer member delete app-pool <member-id>   # drain
# update instance
openstack loadbalancer member create --address <ip> --protocol-port 8080 app-pool
```

---

## Gotchas

1. **Cinder volume AZ must match instance AZ** — attach fails at apply time with a generic error, not at plan time. Use a single Terraform variable for both resources.
2. **Floating IP quota is tight** — attach floating IPs to the load balancer only, not individual instances. One floating IP for the whole 3-tier app.
3. **Neutron router needs two attachments** — `set --external-gateway` (uplink to admin network) AND `add subnet` (downlink to your subnet). Missing either one means no routing.
4. **Flavor `disk = 0` requires boot-from-volume** — launching with such a flavor and no `--boot-from-volume` flag will error. Know your org's flavors before writing Terraform.
5. **Token scope is per-project** — if your CI pipeline needs to act on two projects, it needs two separate `clouds.yaml` entries and two auth calls.
6. **LB must be ACTIVE before floating IP attach** — Octavia LB creation is async. Poll `openstack loadbalancer show` for ACTIVE state before associating the floating IP, or use `depends_on` in Terraform.
7. **Ask your admin before you start** — external network name, AZ names, available flavors, and your quota. `openstack availability zone list`, `openstack flavor list`, `openstack quota show`.

---

## Quick Reference

**Essential commands:**

```text
openstack quota show                          # check vCPU, RAM, floating IP limits
openstack availability zone list              # get AZ names
openstack flavor list                         # get flavor names and sizes
openstack image list                          # get available OS images
openstack network list                        # find external network name
openstack server list                         # your running instances
openstack loadbalancer list                   # LB status (wait for ACTIVE)
openstack floating ip list                    # floating IPs in your project
```

**CI/CD pipeline roles:**

| Role | Use for |
|---|---|
| `member` | Deploy pipelines (create/delete resources) |
| `reader` | Audit/drift-detection pipelines (read-only) |

**Storage decision:**

| Need | Use |
|---|---|
| VM disk (persistent) | Cinder |
| Build artifacts | Swift |
| DB backups | Swift |
| Shared across VMs | Manila/NFS |

---

## Flashcards

**Q: What problem does OpenStack solve?**
A: It gives organisations AWS-like self-service infrastructure primitives (compute, network, storage, LB) on hardware they control, with a multi-tenant project model for team isolation.

**Q: What is a Project in OpenStack and how does it differ from an AWS account?**
A: A Project is your isolation boundary — lighter-weight than an AWS account (more like a K8s namespace), with explicit quotas, a shared underlying control plane, and a single user can belong to multiple projects simultaneously.

**Q: What does Keystone give you, and why does your code never re-send the password?**
A: Keystone exchanges your credentials + project name for a short-lived scoped token. All subsequent API calls use the token — other services validate it with Keystone without seeing your password.

**Q: What is the biggest difference between AWS VPC and OpenStack Neutron?**
A: In AWS, the router and internet gateway routing are implicit. In Neutron you explicitly create a router, attach it to the admin-managed external network, and wire your subnet to it. Same end result, more visible plumbing.

**Q: What breaks if a Cinder volume and its instance are in different AZs?**
A: The volume attachment fails at apply time (not plan time) with a generic error. Cinder volumes are physically on storage nodes in a specific AZ and can't be attached to hypervisors in a different AZ.

**Q: Why does only the load balancer get a floating IP in a 3-tier app?**
A: Floating IPs are a limited quota resource and internet exposure is a security risk. The LB is the only public entry point — app and DB servers stay on private IPs, accessed only via the LB or a bastion.

**Q: What is the correct provisioning order for a 3-tier app on OpenStack?**
A: Network/subnet/router → security groups → keypair → DB instance + Cinder volume → app instances → load balancer + pool + members → floating IP on LB VIP.

**Q: When does Octavia's LB become attachable to a floating IP?**
A: Only when its status is ACTIVE. Creation is async — poll `openstack loadbalancer show` or use Terraform's implicit `depends_on` before associating the floating IP.

---

## Session Handoff Prompt

> I know OpenStack well as a tenant consumer. Mental model: OpenStack = collection of named services (Nova=compute, Neutron=networking, Cinder=block storage, Swift=object storage, Octavia=LB, Keystone=auth) with a Project as the isolation boundary. Auth via clouds.yaml → Keystone token → scoped API calls. Networking requires explicit router assembly (network → subnet → router → external gateway → subnet attachment → floating IP on LB only). Cinder volumes must match instance AZ. Provisioning order: network → security groups → keypair → DB+volume → app instances → LB → floating IP. I'm building: [describe your task here].
