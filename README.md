# Ansible AWS Web Infrastructure

Provisions a multi-AZ web tier on AWS — VPC, subnets, security groups, EC2 instances, and an Application Load Balancer — then configures Apache on each instance using Ansible. The full stack deploys and tears down with a single command.

---

## Architecture

```
                        ┌─────────────────────────────────────┐
                        │         VPC  10.15.0.0/16           │
                        │         Internet Gateway             │
                        │         Public Route Table           │
                        │                                      │
                        │   ┌──────────────────────────────┐  │
                        │   │  Application Load Balancer   │  │
                        │   │  internet-facing  port 80    │  │
                        │   └───────────┬──────────────────┘  │
                        │               │                      │
                        │    ┌──────────┴──────────┐          │
                        │    ▼                     ▼          │
                        │ ┌──────────────┐  ┌──────────────┐  │
                        │ │  EC2 × 2     │  │  EC2 × 2     │  │
                        │ │ us-east-2a   │  │ us-east-2b   │  │
                        │ │ 10.15.0.0/28 │  │10.15.0.16/28 │  │
                        │ └──────────────┘  └──────────────┘  │
                        └─────────────────────────────────────┘
```

**Resources provisioned:**

| Resource | Detail |
|---|---|
| VPC | `10.15.0.0/16`, two public subnets across `us-east-2a` and `us-east-2b` |
| Internet Gateway | Attached to VPC with a public route table |
| Security Groups | One for instances (80, 443, SSH restricted), one for the ALB (80) |
| EC2 Instances | 4 × `t2.micro` Ubuntu — 2 per subnet |
| Target Group | HTTP port 80, instance type, session stickiness enabled |
| ALB | Internet-facing, HTTP port 80, forwards to target group |
| Apache | Installed and configured on all instances via Ansible |

---

## Project Structure

```
.
├── deploy.yml          # Orchestrator — runs aws.yml then webserver.yaml
├── destroy.yml         # Tears down all provisioned resources in dependency order
├── aws.yml             # Provisions all AWS infrastructure
├── webserver.yaml      # Configures Apache on EC2 instances
├── vars.yml            # All configurable variables — edit before running
├── index_html.j2       # Jinja2 template for instance index page
├── aws_ec2.yml         # AWS dynamic inventory plugin config
└── inventory.ini       # Reserved for static inventory use
```

---

## Prerequisites

- Ansible `>= 2.12`
- Python `>= 3.9`
- AWS collections installed:

```bash
ansible-galaxy collection install amazon.aws community.aws community.general
```

- AWS credentials configured (`~/.aws/credentials` or environment variables)
- An EC2 key pair created in `us-east-2` and the `.pem` file available locally

---

## Configuration

Edit `vars.yml` before running. Variables marked **Required** must be set before the playbook will work:

| Variable | Description | Default | Change? |
|---|---|---|---|
| `region` | AWS region to deploy into | `us-east-2` | Optional |
| `project_name` | Name prefix applied to ALB and target group | `ans-aws-alb` | Optional |
| `ami_id` | Ubuntu AMI ID — must match your region | `ami-0f5fcdfbd140e4ab7` | If changing region |
| `key_name` | EC2 key pair name as it appears in AWS | `your_key_name` | **Required** |
| `ssh_key_path` | Local path to the `.pem` file | `~/your_key.pem` | **Required** |
| `ssh_ip` | CIDR permitted to SSH to instances | `0.0.0.0/0` | **Recommended** |
| `subnet.one` | AZ for first subnet | `us-east-2a` | Optional |
| `subnet.two` | AZ for second subnet | `us-east-2b` | Optional |

> **Note on `ssh_ip`:** The default allows SSH from anywhere. Set this to your own IP (`x.x.x.x/32`) before deploying.

> **Note on `ami_id`:** The default AMI is region-specific to `us-east-2`. If you change `region`, look up the equivalent Ubuntu AMI in the [AWS AMI catalogue](https://cloud-images.ubuntu.com/locator/ec2/) for that region.

---

## Usage

### Full deploy

```bash
ansible-playbook deploy.yml
```

Runs `aws.yml` then `webserver.yaml` in sequence. EC2 instance IPs are registered into the `aws` inventory group dynamically at runtime via `add_host` — no static inventory file needed.

### Infrastructure only

```bash
ansible-playbook aws.yml
```

### Web server configuration only

Requires instances to already be running and reachable.

```bash
ansible-playbook webserver.yaml -i <your-inventory>
```

### Destroy everything

```bash
ansible-playbook destroy.yml
```

Resources are removed in dependency order: instances → ALB → target group → security groups → route table → IGW → subnets → VPC. Lookup is tag-based so no resource IDs need to be known in advance.

---

## How It Works

**Infrastructure — `aws.yml`**

1. Creates VPC, two public subnets, IGW, and a public route table with default IPv4 and IPv6 routes
2. Creates two security groups — one scoped to the ALB (port 80), one to the instances (80, 443, SSH restricted to `ssh_ip`)
3. Launches 4 EC2 instances with `wait: true`, then polls each on port 22 before proceeding
4. Adds all instance IPs to the `aws` in-memory host group with SSH connection details
5. Creates the ALB target group, registers all instances individually, then provisions the ALB

**Configuration — `webserver.yaml`**

1. Updates the package cache and installs Apache, `unzip`, and `wget`
2. Ensures Apache is started and enabled; sets `/var/www/html` ownership to `www-data`
3. Opens ports 22 and 80 via UFW
4. Renders `index_html.j2` to `/var/www/html/index.html` and triggers an Apache restart via handler
5. Confirms Apache is responding locally with an HTTP check against `127.0.0.1`

---

## Known Limitations & Future Improvements

This project demonstrates core Ansible and AWS provisioning patterns. The following are intentional simplifications that would need to be addressed before any production use.

**Security**
- `StrictHostKeyChecking=no` is used in `add_host` to avoid interactive prompts during provisioning. Production environments should use SSM Session Manager to remove SSH exposure entirely, or manage known hosts properly
- No `ansible-vault` — `ssh_key_path` and `ssh_ip` are in plaintext `vars.yml`. These should be vaulted or injected via environment variables in any shared or CI context
- No HTTPS — the ALB listener is HTTP only. A production deployment would attach an ACM certificate, add a port 443 listener, and redirect HTTP traffic to HTTPS

**Architecture**
- Static EC2 instances are used intentionally here to demonstrate the `add_host` runtime inventory pattern. In production, an Auto Scaling Group backed by a Launch Template is the correct pattern — it replaces unhealthy instances automatically and scales with load
- Both subnets are public. Instances should sit in private subnets behind a NAT Gateway, with only the ALB exposed to the internet

**Operational**
- No `requirements.yml` — collection versions are unpinned and can drift between runs
- The AMI ID is hardcoded and region-specific. A lookup via `amazon.aws.ec2_ami_info` or a Packer-built AMI would make the project portable across regions
- Session stickiness is enabled on the target group but `stickiness_type` is not explicitly declared — this should be set to `lb_cookie` or `app_cookie` with a matching cookie name to avoid ambiguous API behaviour

---
