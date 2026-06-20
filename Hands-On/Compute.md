# AWS Hands-On Lab Guide — Console Practice

Two labs matching your two classes. Do these step-by-step directly in the AWS Console (not Terraform) so students build the mental model first. **Use a free-tier account and tear everything down at the end (cleanup steps included) to avoid charges.**

---

# LAB 1 (Class 1): Network Foundation + Public EC2

### Goal
Build a VPC with public/private subnets, NACLs, a Security Group, and launch an EC2 instance you can SSH into.

### Step 1 — Create the VPC
1. Console → **VPC** → **Your VPCs** → **Create VPC**
2. Choose "VPC and more" (this auto-creates subnets, route tables, IGW)
3. Name: `lab-vpc`, IPv4 CIDR: `10.0.0.0/16`
4. Number of AZs: 2, Public subnets: 2, Private subnets: 2
5. NAT gateways: **None** for now (we add this in Lab 2)
6. Click **Create VPC**

Have students screenshot the resource map it generates — this is the diagram coming to life.

### Step 2 — Inspect what got created
1. **Subnets**: note which are public (auto-assign public IP = yes) vs private
2. **Route Tables**: open the public route table → Routes tab → show `0.0.0.0/0 → igw-xxxx`
3. **Internet Gateway**: show it's attached to `lab-vpc`

### Step 3 — Create a Network ACL (stateless layer)
1. VPC → **Network ACLs** → **Create network ACL**
2. Name: `lab-public-nacl`, VPC: `lab-vpc`
3. Associate it with your public subnet
4. Add inbound rule: rule 100, type SSH (22), source `0.0.0.0/0` (or your IP), ALLOW
5. Add inbound rule: rule 110, type HTTP (80), source `0.0.0.0/0`, ALLOW
6. Add a high-numbered ephemeral port rule (1024–65535) for return traffic — explain **why** (stateless = need explicit return rule, unlike SGs)

### Step 4 — Create a Security Group (stateful layer)
1. VPC → **Security Groups** → **Create security group**
2. Name: `lab-web-sg`
3. Inbound: SSH (22) from your IP, HTTP (80) from `0.0.0.0/0`
4. Outbound: leave default (all allowed)
5. Point out: no need for a return-traffic rule here — SGs are stateful

### Step 5 — Launch EC2 in the public subnet
1. EC2 → **Launch instance**
2. Name: `lab-web-server`, AMI: Amazon Linux 2023, type: t2.micro
3. Key pair: create new, download the `.pem`
4. Network: `lab-vpc`, Subnet: the **public** subnet
5. Auto-assign public IP: **Enable**
6. Security group: select `lab-web-sg`
7. Launch

### Step 6 — Look at the ENI
1. EC2 → select your instance → **Networking** tab → click the Elastic Network Interface ID
2. Show: private IP, public IP, the SG attached to the ENI (not the instance — the SG lives on the ENI)

### Step 7 — Connect
1. SSH in: `ssh -i lab-key.pem ec2-user@<public-ip>`
2. Or use **EC2 Instance Connect** (browser-based) — easiest for a classroom
3. Run `sudo yum install -y httpd && sudo systemctl start httpd && echo "Hello from Lab 1" | sudo tee /var/www/html/index.html`
4. Open the public IP in a browser → confirm the page loads (this proves SG + NACL + route table + IGW all work together)

**End of Lab 1 — don't delete yet, Lab 2 builds on this.**

---

# LAB 2 (Class 2): Private EC2 + Load Balancer + Route 53

### Goal
Add a private EC2 instance, a NAT Gateway so it can reach the internet, an Application Load Balancer in front, and a Route 53 record pointing to it.

### Step 1 — Add a NAT Gateway
1. VPC → **NAT Gateways** → **Create NAT gateway**
2. Subnet: your **public** subnet
3. Allocate an Elastic IP
4. Create
5. VPC → **Route Tables** → select the **private** route table → Routes → Edit → add `0.0.0.0/0 → nat-xxxx`

### Step 2 — Launch EC2 in the private subnet
1. EC2 → Launch instance, name: `lab-app-server`
2. Subnet: the **private** subnet, auto-assign public IP: disabled (it's private — no choice anyway)
3. Security group: create `lab-app-sg` — inbound HTTP (80) **from `lab-web-sg`** only, not `0.0.0.0/0` (key teaching point: private tier only trusts the public tier)
4. Launch
5. Note: to SSH into this one, you'd need a bastion host or SSM Session Manager — mention this as the real-world pattern

### Step 3 — Create an Application Load Balancer
1. EC2 → **Load Balancers** → **Create load balancer** → Application Load Balancer
2. Name: `lab-alb`, Scheme: internet-facing
3. VPC: `lab-vpc`, Mappings: select both **public** subnets (ALB needs 2 AZs)
4. Security group: create new `lab-alb-sg` — inbound HTTP (80) from `0.0.0.0/0`
5. Listener: HTTP:80 → forward to a new target group

### Step 4 — Target group
1. Create target group: `lab-app-tg`, target type: instances, protocol HTTP:80
2. Health check path: `/`
3. Register your `lab-app-server` (private instance) as the target
4. Update `lab-app-sg` inbound rule to allow port 80 from `lab-alb-sg` instead of `lab-web-sg` (cleaner: ALB is now the only thing allowed to reach the app tier)

### Step 5 — Test the ALB
1. Wait for target health to show "healthy"
2. Copy the ALB's DNS name (e.g. `lab-alb-123456.us-east-1.elb.amazonaws.com`)
3. Paste into browser — should reach the private instance through the ALB

### Step 6 — Route 53
1. Route 53 → **Hosted zones** → if students have a domain, create a hosted zone; otherwise create a **private hosted zone** for practice, or use Route 53 with a registered test domain
2. Create record: type **A**, Alias: yes → Alias to Application Load Balancer → select region → select `lab-alb`
3. Record name: `app.yourdomain.com`
4. Save, then `dig app.yourdomain.com` or browse to it once DNS propagates

### Step 7 — Trace the full path (whiteboard moment)
Walk the request: **User → Route 53 (DNS) → ALB (public subnet, alb-sg) → Target Group → EC2 in private subnet (app-sg) → response back out.**

---

## Cleanup (do this same day to avoid charges)
1. Delete the Route 53 record (and hosted zone if created just for this)
2. Delete the ALB, then the target group
3. Terminate both EC2 instances
4. Delete the NAT Gateway (this is the one that costs money hourly — don't skip it)
5. Release the Elastic IP
6. Delete the VPC (this cascades subnets, route tables, IGW, NACLs, SGs)

---

## Suggested class split
- **Class 1**: Steps in Lab 1 only (VPC, subnets, NACL, SG, ENI, public EC2) — ends with a working public web server
- **Class 2**: Lab 2 (NAT, private EC2, ALB, target groups, Route 53) — ends with the full architecture, then cleanup, then show the equivalent Terraform side-by-side so they see console action ↔ HCL resource

Want me to put together a short "diagram → console click-path → Terraform resource" cheat sheet as a one-pager for students to keep?
