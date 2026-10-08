# Phase 1: Provision and Configure AWS Servers with Terraform and Ansible

Terraform creates the AWS infrastructure. Ansible then configures it: it updates the server,
installs Docker, starts Docker, and proves the server works by running test containers.

```
Laptop (VS Code + Terraform)
   │  terraform apply
   ▼
AWS ── VPC 10.0.0.0/16 ── public subnet 10.0.1.0/24
        ├── Control node  10.0.1.10  (Ansible installed, SSH from my IP only)
        └── Managed node  10.0.1.11  (Docker installed by Ansible, port 80 open)
                ▲
                └── SSH from the control node only
```

| Tool | Runs on | Job |
|---|---|---|
| Terraform | Laptop | Builds the VPC, subnet, security groups, IAM role and two EC2 instances |
| Ansible | Control node (EC2) | Configures the managed node: updates it, installs and starts Docker, tests it |
| VS Code Remote-SSH | Laptop | Lets you edit and run everything on the control node from your editor |

## Repository structure

```
.
├── README.md
├── .gitignore                  keeps keys, state and secrets out of Git
├── terraform/
│   ├── main.tf                 all AWS resources
│   ├── variables.tf            inputs (region, CIDRs, instance type, key path, my IP)
│   ├── outputs.tf              IPs and the SSH config snippet
│   └── terraform.tfvars.example   template for your own values
├── ansible/
│   ├── ansible.cfg             Ansible settings
│   ├── inventory.ini           which servers Ansible manages
│   ├── docker.yml              the playbook that installs Docker
│   └── run.sh                  helper script: checks, tests, then runs the playbook
└── screenshots/                evidence for submission
```

## Prerequisites (on your laptop)

- An AWS account and an IAM user (not root) with permissions for EC2, VPC and IAM
- AWS CLI configured: `aws configure`
- Terraform 1.5 or newer
- VS Code with the **Remote - SSH** extension
- Git

## Step by step

### Step 1: Create an SSH key
```
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```
This creates a private key (stays on your laptop, never commit it) and a public key
(`.pub`, which Terraform uploads to AWS so you can log in).

Load the private key into the SSH agent. The control node will use it (through agent
forwarding) to reach the managed node, so the private key is never copied to a server.
- macOS/Linux: `eval "$(ssh-agent -s)"` then `ssh-add ~/.ssh/id_ed25519`
- Windows (PowerShell as Administrator, once): `Get-Service ssh-agent | Set-Service -StartupType Automatic; Start-Service ssh-agent`, then `ssh-add $HOME\.ssh\id_ed25519`

### Step 2: Configure Terraform variables
```
cd terraform
cp terraform.tfvars.example terraform.tfvars
```
Edit `terraform.tfvars`:
```hcl
region          = "eu-west-1"
my_ip_cidr      = "YOUR.PUBLIC.IP/32"            # find it: curl ifconfig.me
public_key_path = "~/.ssh/id_ed25519.pub"        # Windows: "C:/Users/<you>/.ssh/id_ed25519.pub"
```
`my_ip_cidr` restricts SSH on the control node to your IP. `terraform.tfvars` is git-ignored.

### Step 3: Build the infrastructure
```
terraform init      # downloads the AWS provider
terraform plan      # previews what will be created
terraform apply     # creates it (type "yes")
```
Terraform creates the VPC, subnet, internet gateway, route table, two security groups, an
IAM role for ECR access (used in Phase 2/3), a key pair, the control node and the managed
node. The control node installs Ansible on first boot using `user_data`.

Take a screenshot of the instances running in the EC2 console. Wait about a minute for
Ansible to finish installing.

### Step 4: Connect VS Code to the control node
Terraform prints an `ssh_config_snippet` output. Paste it into `~/.ssh/config`:
```
Host ansible-control
  HostName <control_public_ip>
  User ubuntu
  IdentityFile ~/.ssh/id_ed25519
  ForwardAgent yes
```
In VS Code press `F1`, choose **Remote-SSH: Connect to Host**, and select `ansible-control`.
`ForwardAgent yes` is what lets the control node use your key to reach the managed node.

### Step 5: Get the project onto the control node
In the VS Code terminal (now running on the control node):
```
git clone <your-repo-url>
cd <repo>/ansible
```

### Step 6: Run Ansible
```
bash run.sh
```
The script does five things in order:
1. Confirms Ansible is installed (installs it if missing).
2. Confirms your SSH key was forwarded from the laptop.
3. Pings the managed node (`ansible managed -m ping`) to prove Ansible can connect.
4. Runs a syntax check on the playbook.
5. Runs the playbook `docker.yml`.

You can also run the steps by hand: `ansible managed -m ping` then `ansible-playbook docker.yml`.

### Step 7: What the playbook does (`ansible/docker.yml`)

| Task | Purpose |
|---|---|
| Update apt cache and upgrade packages | Brings the server up to date |
| Install `docker.io` and `python3-docker` | Installs Docker and the library Ansible's Docker module needs |
| Start and enable the Docker service | Starts Docker now and on every reboot |
| Add `ubuntu` to the `docker` group | Allows running `docker` without `sudo` after the next login |
| Check and show the Docker version | Proves Docker is installed |
| Run `hello-world` container | Proves Docker can pull and run images |
| Run an nginx container on port 80 | Gives a visible web page to test |
| Verify HTTP 200 on localhost | Proves the web server actually responds |

### Step 8: Verify
- The playbook ends with `failed=0` in the `PLAY RECAP`. Screenshot it.
- Open `http://<managed_public_ip>` in a browser to see the nginx welcome page. Screenshot it.

### Step 9: Clean up
```
cd terraform && terraform destroy
```
Destroy when you finish to avoid charges.

## Security notes
- SSH to the control node is limited to your IP; the managed node accepts SSH only from the control node.
- Never commit `.pem` files, private keys, `terraform.tfstate` or `terraform.tfvars` (the `.gitignore` covers them).
- Redact public IPs, instance IDs and account IDs from screenshots before committing.
- If your IP changes, update `my_ip_cidr` and run `terraform apply` again.

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `terraform plan` says the key file does not exist | Wrong `public_key_path`. On Windows use `C:/Users/<you>/.ssh/id_ed25519.pub` |
| SSH to the control node times out | `my_ip_cidr` is wrong or your IP changed |
| `ansible: command not found` | `user_data` is still running; wait a minute, or `bash run.sh` installs it |
| `run.sh` says no SSH agent keys | Run `ssh-add` on the laptop and reconnect with `ForwardAgent yes` |
| `UNREACHABLE` when pinging the managed node | Agent forwarding is not working, or the managed node is still booting |
| `AccessDenied` on `iam:` during apply | Your IAM user lacks IAM permissions; add them or remove the IAM resources for now |

## Roadmap
- **Phase 2:** containerize the provided application and push it to Amazon ECR
- **Phase 3:** deploy it to the managed node automatically with GitHub Actions
