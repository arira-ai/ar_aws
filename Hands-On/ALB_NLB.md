## Comprehensive Hands-On Lab: AWS ALB and NLB Mastery

This comprehensive, step-by-step hands-on lab is designed to guide you through building, configuring, and testing both an **Application Load Balancer (ALB)** and a **Network Load Balancer (NLB)** from scratch using the AWS Management Console.

By completing this lab, you will understand how AWS manages traffic distribution across backend servers, handles automatic health checks, routes requests based on application paths, and provides fault tolerance.

---

### Lab Overview

* **Estimated Completion Time:** 90 Minutes
* **Target Audience:** Beginner to Intermediate Cloud Engineering Students / Trainees
* **Cost Estimate:** Less than $0.50 (If cleaned up immediately after completion; utilizes AWS Free Tier eligible resources).

---

### Prerequisites & Initial Setup

Before starting, ensure you have access to an AWS Account with an IAM User possessing administrative or full access permissions for EC2, VPC, CloudWatch, and Elastic Load Balancing.

#### Required Baseline Infrastructure

* **Default VPC:** Ensure your AWS Region has a default VPC configured.
* **Public Subnets:** At least two default public subnets across distinct Availability Zones (AZs) with an attached Internet Gateway.
* **Key Pair:** An existing RSA key pair for optional SSH validation (though we will primarily use EC2 Instance Connect).

---

### Part 1: Architecture Diagrams

#### Application Load Balancer Architecture

```
Internet (HTTP Request to http://<ALB-DNS-Name>/)
      ↓
Application Load Balancer (Layer 7 - Inspects HTTP Headers/Paths)
      ↓
Target Group (Target Type: Instance, Protocol: HTTP, Port: 80)
      ├── WebServer-1 (Availability Zone A - Healthy)
      └── WebServer-2 (Availability Zone B - Healthy)

```

#### Network Load Balancer Architecture

```
Internet (TCP Traffic to http://<NLB-DNS-Name>:80)
      ↓
Network Load Balancer (Layer 4 - Fast, Ultra-low Latency TCP Forwarding)
      ↓
Target Group (Target Type: Instance, Protocol: TCP, Port: 80)
      ├── WebServer-1 (Availability Zone A - Healthy)
      └── WebServer-2 (Availability Zone B - Healthy)

```

---

### Part 2: Create Backend EC2 Instances

We will spin up two identical virtual servers to serve as our application backend targets.

#### Step-by-Step Console Navigation

1. Open the AWS Management Console and navigate to **EC2** by searching for it in the top search bar.
2. In the left navigation pane, click **Instances**, then click the orange **Launch instances** button in the top right.

#### Configuration Settings for WebServer-1

* **Name:** `WebServer-1`
* **Application and OS Images (AMI):** Select **Amazon Linux** (Choose the default *Amazon Linux 2023 AMI*, Free Tier eligible).
* **Instance Type:** `t2.micro` (or `t3.micro` depending on region availability).
* **Key pair (login):** Select an existing key pair or choose *Proceed without a key pair* (since we will use EC2 Instance Connect).
* **Network Settings:** Click **Edit** (top right of the network section):
* **VPC:** Select your Default VPC.
* **Subnet:** Choose the first available public subnet (e.g., `us-east-1a`).
* **Auto-assign public IP:** Select **Enable**.


* **Firewall (Security Groups):** Select **Create security group**.
* **Security group name:** `web-traffic-sg`
* **Description:** Allow inbound HTTP and SSH traffic.
* **Inbound Security Groups Rules:**
* **Rule 1:** Type: `SSH`, Source type: `Anywhere-IPv4` (`0.0.0.0/0`)
* **Rule 2:** Click **Add security group rule**. Type: `HTTP`, Source type: `Anywhere-IPv4` (`0.0.0.0/0`)




* Click **Launch instance**.

#### Configuration Settings for WebServer-2

1. Click **Launch instances** again to create the second server.
2. **Name:** `WebServer-2`
3. **Application and OS Images (AMI):** **Amazon Linux 2023 AMI**
4. **Instance Type:** `t2.micro`
5. **Network Settings:** Click **Edit**:
* **VPC:** Default VPC.
* **Subnet:** Choose a *different* public subnet than the first server (e.g., `us-east-1b`) to ensure high availability.
* **Auto-assign public IP:** **Enable**.



* **Firewall (Security Groups):** Select **Select existing security group** and choose the `web-traffic-sg` you created for the first instance.

6. Click **Launch instance**.

> **Verification Checkpoint:** Navigate back to the **Instances** view. Wait 1–2 minutes until both `WebServer-1` and `WebServer-2` show an **Instance state** of `Running` and **Status check** displays `2/2 checks passed`.

---

### Part 3: Install and Configure Apache Web Server

To test traffic distribution, each instance must serve a unique webpage stating its own name.

#### Step-by-Step Installation via EC2 Instance Connect

1. In the EC2 Instances view, select the checkbox next to **WebServer-1**.
2. Click the **Connect** button at the top of the console page.
3. Choose the **EC2 Instance Connect** tab, ensure the username is `ec2-user`, and click **Connect**. A new browser tab containing a Linux terminal will open.
4. Run the following commands in sequence to update packages, install Apache, customize the index page, and start the service:

```bash
sudo dnf update -y
sudo dnf install httpd -y
echo "<h1>Welcome from WebServer-1</h1>" | sudo tee /var/www/html/index.html
sudo systemctl start httpd
sudo systemctl enable httpd

```

5. Close the tab, go back to the EC2 instances dashboard, select **WebServer-2**, and click **Connect**.
6. Run the exact same commands, but modify the webpage text content to match the second server:

```bash
sudo dnf update -y
sudo dnf install httpd -y
echo "<h1>Welcome from WebServer-2</h1>" | sudo tee /var/www/html/index.html
sudo systemctl start httpd
sudo systemctl enable httpd

```

#### Verification Checkpoint

Go back to the EC2 Instances console page. Copy the **Public IPv4 address** for `WebServer-1`, open a new browser tab, and navigate to `http://<WebServer-1-Public-IP>`. You should see a page displaying "Welcome from WebServer-1". Repeat this process for `WebServer-2` to confirm both web servers are fully functional on the internet.

---

### Part 4: Create the HTTP Target Group

Target groups route incoming requests from a load balancer to one or more registered backend targets.

#### Step-by-Step Console Navigation

1. In the left navigation menu of the EC2 console, scroll down to the **Load Balancing** section and click **Target Groups**.
2. Click the orange **Create target group** button.

#### Configuration Settings

* **Choose a target type:** Select **Instances**.
* **Target group name:** `tg-http-alb`
* **Protocol:** `HTTP`
* **Port:** `80`
* **IP address type:** `IPv4`
* **VPC:** Select your Default VPC.
* **Protocol version:** `HTTP1`
* **Health checks:** Expand this section to view the metrics.
* **Health check protocol:** `HTTP`
* **Health check path:** `/`


* **Advanced health check settings:** (Expand to modify intervals)
* **Healthy threshold:** `3` (Consecutive successful checks required to mark it healthy)
* **Unhealthy threshold:** `2` (Consecutive failed checks required to mark it unhealthy)
* **Timeout:** `5 seconds`
* **Interval:** `15 seconds` (Speeds up deployment visualization for this lab environment)


* Click **Next**.

#### Register Targets

1. In the **Available instances** table below, select the checkboxes for both **WebServer-1** and **WebServer-2**.
2. In the **Ports for selected instances** field, verify it says `80`.
3. Click the **Include as pending below** button.
4. Verify both instances now appear in the **Targets** table with a status of `Pending allocation`.
5. Click **Create target group**.

---

### Part 5: Create and Configure the Application Load Balancer (ALB)

The ALB will operate at the Application Layer (Layer 7) to inspect incoming HTTP requests and balance them across our target group.

#### Step-by-Step Console Navigation

1. In the left navigation pane under **Load Balancing**, click **Load Balancers**.
2. Click **Create load balancer** (top right).
3. Under **Application Load Balancer**, click **Create**.

#### Configuration Settings

* **Load balancer name:** `alb-web-demo`
* **Scheme:** Select **Internet-facing** (This allows the load balancer to accept public traffic over the internet).
* **IP address type:** `IPv4`
* **Network mapping:**
* **VPC:** Select your Default VPC.
* **Mappings (Availability Zones):** Check the boxes for at least **two** Availability Zones (e.g., `us-east-1a` and `us-east-1b`). You must select the specific subnets where your EC2 instances reside.


* **Security groups:** Remove the default security group selected by clicking the 'X'. Select your existing **web-traffic-sg** from the drop-down list so the ALB can accept inbound HTTP port 80 requests.
* **Listeners and routing:**
* **Protocol:** `HTTP`
* **Port:** `80`
* **Default action:** Select **Forward to** and pick your created target group: `tg-http-alb`.


* Scroll to the bottom and click **Create load balancer**.
* Click **View load balancers**.

> **Internal Mechanism Note:** While the ALB status shows `Provisioning`, AWS is spawning internal managed node instances across the selected availability zones and attaching elastic network interfaces (ENIs) to route your public web traffic.

---

### Part 6: Test and Verify the ALB

#### Initial Status Verification

1. Wait 2–3 minutes for the `alb-web-demo` state to change from `Provisioning` to `Active`.
2. Select your new load balancer and look at the **Details** tab below. Locate and copy the long **DNS name** (e.g., `alb-web-demo-123456789.us-east-1.elb.amazonaws.com`).

#### Verification via Web Browser

1. Open a fresh browser window, paste the copied ALB DNS name into the address bar, and press Enter: `http://<ALB-DNS-Name>`
2. You will see either "Welcome from WebServer-1" or "Welcome from WebServer-2".
3. Refresh the page (`Ctrl+R` or `Cmd+R`) multiple times. You will see the text alternate between **WebServer-1** and **WebServer-2**.

#### Verification via Command Line (Curl)

To view this behavior systematically without browser caching issues, open a local terminal or command prompt window and execute the following loop command (replace with your actual ALB DNS name):

```bash
for i in {1..6}; do curl -s http://alb-web-demo-123456789.us-east-1.elb.amazonaws.com | grep "Welcome"; done

```

#### Expected Output

```text
<h1>Welcome from WebServer-1</h1>
<h1>Welcome from WebServer-2</h1>
<h1>Welcome from WebServer-1</h1>
<h1>Welcome from WebServer-2</h1>
<h1>Welcome from WebServer-1</h1>
<h1>Welcome from WebServer-2</h1>

```

#### Technical Explanation

The ALB implements a standard **Round-Robin routing algorithm** by default. Because it acts as a Layer 7 proxy, it terminates the incoming client connection, inspects the HTTP request layer, and establishes a separate network connection to distribute requests evenly among healthy backend targets.

---

### Part 7: Demonstrate Health Checks and Self-Healing

#### Simulating a Server Outage

1. Navigate back to the **EC2 Console** and click **Instances**.
2. Select **WebServer-1**, click **Instance state** at the top right, and choose **Stop instance**. Confirm by clicking **Stop**.

#### Observing Target Health Degradation

1. In the left navigation menu, go to **Target Groups** and click on `tg-http-alb`.
2. Click the **Targets** tab in the lower panel.
3. Watch the status of `WebServer-1`. Within 30 seconds (based on our 15-second health check interval), its status will change from `Healthy` to `Unhealthy` with an execution description of *Target.ResponseCodeMismatch* or *Target.FailedHealthChecks*.

#### Testing User Experience During Outage

Open your terminal or browser and refresh your connection to the ALB DNS name multiple times:

```bash
curl http://alb-web-demo-123456789.us-east-1.elb.amazonaws.com

```

Notice that **100% of the traffic** now seamlessly hits `WebServer-2`. The ALB automatically detects the health drop-off and stops routing requests to the dead node, ensuring zero downtime for your users.

#### Recovery Actions

1. Go back to **Instances**, select **WebServer-1**, and click **Instance state** → **Start instance**.
2. Return to the **Target Groups** → `tg-http-alb` targets pane.
3. Observe the instance cycle from `Unhealthy` to `Initial` (as health checks run), and finally back to a green `Healthy` status.
4. Re-run your browser or curl tests; the load balancer will automatically resume round-robin distribution to both instances.

---

### Part 8: Create and Configure a Network Load Balancer (NLB)

Unlike an ALB, a Network Load Balancer operates at Layer 4 (Transport Layer). It is designed to route raw TCP, UDP, or TLS streams at extreme scale with ultra-low latencies.

#### Step 1: Create a Layer 4 Target Group

1. Go to **Target Groups** in the EC2 menu and click **Create target group**.
2. **Target type:** `Instances`
3. **Target group name:** `tg-tcp-nlb`
4. **Protocol:** `TCP`
5. **Port:** `80`
6. **VPC:** Select your Default VPC.
7. Click **Next**.
8. Select both **WebServer-1** and **WebServer-2**, click **Include as pending below**, and click **Create target group**.

#### Step 2: Provision the Network Load Balancer

1. Go to **Load Balancers** and click **Create load balancer**.
2. Under **Network Load Balancer**, click **Create**.
3. **Load balancer name:** `nlb-tcp-demo`
4. **Scheme:** `Internet-facing`
5. **Network mapping:**
* **VPC:** Select your Default VPC.
* **Mappings:** Check at least two Availability Zones (the same zones housing your instances). Note that you have the option here to pin a **Static Elastic IP address** to each subnet interface—a unique capability exclusive to NLBs. For this lab, keep it set to *AWS-assigned IP*.


6. **Security groups:** Select your existing `web-traffic-sg` from the dropdown list.
7. **Listeners and routing:**
* **Protocol:** `TCP`
* **Port:** `80`
* **Default action:** Forward to `tg-tcp-nlb`.


8. Click **Create load balancer**.

---

### Part 9: Test and Verify the NLB

1. Wait for the `nlb-tcp-demo` status to transition to **Active**.
2. Copy the **DNS name** from the details page (e.g., `nlb-tcp-demo-12345678.elb.us-east-1.amazonaws.com`).
3. Execute a curl check or navigate to the address in your browser:

```bash
curl -s http://nlb-tcp-demo-12345678.elb.us-east-1.amazonaws.com | grep "Welcome"

```

#### Crucial Behavioral Differences

* **Sticky Connections:** When testing an NLB using a standard web browser, you might notice that refreshing the page repeatedly returns the exact same web server instead of alternating instantly.
* **Why does this happen?** The NLB targets traffic at the Transport layer using a hashing algorithm based on the protocol, source IP address, and source port. Since browsers keep underlying TCP connections open to minimize latency, all subsequent HTTP page requests over that active socket are sent to the exact same backend server.

---

### Part 10: Comparison Reference Matrix: ALB vs. NLB

| Feature / Criteria | Application Load Balancer (ALB) | Network Load Balancer (NLB) |
| --- | --- | --- |
| **OSI Operating Layer** | Layer 7 (Application) | Layer 4 (Transport) |
| **Supported Protocols** | HTTP, HTTPS, gRPC, WebSockets | TCP, UDP, TLS |
| **Routing Decisions** | Based on URLs, Paths, HTTP Headers, Cookies | Based on Network Layer IP and Port hashes |
| **Static / Elastic IP** | No (IP addresses change dynamically) | Yes (Provides fixed static IPs per Subnet/AZ) |
| **Path / Host Routing** | Yes (Built-in context rule evaluation) | No |
| **Performance Scale** | Millions of requests/sec (Scales gradually) | Tens of millions of requests/sec (Instant spikes) |
| **Average Latency** | ~10ms to 100ms (Due to parsing packets) | Ultra-low sub-millisecond latencies (<10ms) |
| **Primary Use Cases** | Microservices, standard web apps, containers | High-performance gaming, trading engines, non-HTTP apps |

---

### Part 11: Demonstrate ALB Listener Rules (Path-Based Routing)

ALBs can route traffic to different target groups based on the structural path of the incoming URL request string.

#### Step 1: Create a Subfolder App Structure on the Instances

To demonstrate path-based routing, let's configure distinct endpoints on our web servers.

1. Connect to **WebServer-1** via EC2 Instance Connect and create a subdirectory named `app1`:

```bash
sudo mkdir -p /var/www/html/app1
echo "<h1>This is Application 1 running on WebServer-1</h1>" | sudo tee /var/www/html/app1/index.html

```

2. Connect to **WebServer-2** via EC2 Instance Connect and create a subdirectory named `app2`:

```bash
sudo mkdir -p /var/www/html/app2
echo "<h1>This is Application 2 running on WebServer-2</h1>" | sudo tee /var/www/html/app2/index.html

```

#### Step 2: Create Separate Target Groups

1. Navigate to **Target Groups** and create a new target group called `tg-app1`. Set protocol to `HTTP`, port `80`, and register **WebServer-1** exclusively.
2. Create another target group called `tg-app2`. Set protocol to `HTTP`, port `80`, and register **WebServer-2** exclusively.

#### Step 3: Configure Path Rules on the ALB Listener

1. Go to **Load Balancers**, select `alb-web-demo`, and click the **Listeners and rules** tab.
2. Select the checkbox next to the existing **HTTP:80** listener, and click **Manage listener** → **View/edit rules**.
3. Click the **Add rules** icon (the plus symbol) or click **Insert rule**.
4. Define **Rule 1**:
* Click **Add condition** → select **Path**.
* Value: `/app1*`
* Click **Add action** → select **Forward to** → choose target group `tg-app1`.
* Click **Save**.


5. Define **Rule 2**:
* Click **Add condition** → select **Path**.
* Value: `/app2*`
* Click **Add action** → select **Forward to** → choose target group `tg-app2`.
* Click **Save**.



#### Verification Checkpoint

Open your browser and append the path extensions to your ALB URL:

* Navigate to `http://<ALB-DNS-Name>/app1/` $\rightarrow$ Displays: *"This is Application 1 running on WebServer-1"*
* Navigate to `http://<ALB-DNS-Name>/app2/` $\rightarrow$ Displays: *"This is Application 2 running on WebServer-2"*

---

### Part 12: Comprehensive Troubleshooting Guide

| Symptom | Probable Root Cause | Verification & Remediation Actions |
| --- | --- | --- |
| **Health Check Failed (Targets show Unhealthy)** | Apache application service stopped or misconfigured security groups. | 1. Connect via EC2 Instance Connect and confirm the service is active by running `sudo systemctl status httpd`. <br>

<br>2. Ensure the instance's Security Group explicitly permits inward traffic on port 80 from the Load Balancer's subnet block. |
| **HTTP 502 Bad Gateway** | The ALB received an invalid response or cannot establish a socket connection to the application server. | 1. Confirm your backend application server is configured to listen exactly on the port specified in the Target Group (Port 80). <br>

<br>2. Verify that your server script doesn't close connections abruptly. |
| **HTTP 503 Service Unavailable** | No healthy backend target instances are currently registered within the corresponding target group. | 1. Navigate to Target Groups, click the targets pane, and verify the health checks are passing.<br>

<br>2. Confirm the ALB configuration hasn't been decoupled from the appropriate target group. |
| **Request Timeout / Connection Hanging** | Network access control issues (Security Groups or NACLs) are blocking traffic paths. | 1. Confirm the Load Balancer's assigned security group permits outbound traffic to the backend instances on Port 80.<br>

<br>2. Verify the VPC's Network Access Control Lists (NACLs) allow traffic on ephemeral ports (1024-65535). |

---

### Part 13: Orderly Resource Cleanup

To prevent unnecessary account billing charges, delete resources in the exact order specified below. This prevents "Resource In Use" or dependency errors.

1. **Load Balancers:** Select both `alb-web-demo` and `nlb-tcp-demo` inside the Load Balancers view. Click **Actions** $\rightarrow$ **Delete load balancer**. (This terminates the load balancer nodes and releases their connected ENIs).
2. **Target Groups:** Once the load balancers are deleted, navigate to **Target Groups**. Select `tg-http-alb`, `tg-tcp-nlb`, `tg-app1`, and `tg-app2`. Click **Actions** $\rightarrow$ **Delete**.
3. **EC2 Instances:** Navigate to **Instances**. Select both `WebServer-1` and `WebServer-2`. Click **Instance state** $\rightarrow$ **Terminate instance**.
4. **Security Groups:** Once the instances are terminated, go to **Security Groups**, select `web-traffic-sg`, and click **Delete security group**.

---

### Final Lab Completion Checklist

Before considering this lab complete, ensure you have successfully completed the following tasks:

* [ ] Both EC2 backend instances show a status of `Running` and respond to public browser requests.
* [ ] The Application Load Balancer successfully alternates traffic between WebServer-1 and WebServer-2 using a round-robin format.
* [ ] Stopping a server triggers an automatic health check failure, and the ALB dynamically routes all user traffic to the surviving node without downtime.
* [ ] The Network Load Balancer successfully routes raw TCP Layer 4 traffic to active targets.
* [ ] All provisioned cloud resources have been terminated in the correct order to avoid charges.

---

### Interactive Architecture & Traffic Flow Simulator

To help solidify how traffic passes through these load balancers under various routing configurations, health statuses, and network layers, use the interactive simulation tool below. You can adjust configurations dynamically to see how traffic flow changes.
