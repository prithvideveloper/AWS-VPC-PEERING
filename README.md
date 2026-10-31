AWS VPC Peering & Private Connectivity Project

📌 Project Overview

This project demonstrates the design, deployment, and testing of a private, non-transitive network connection between two Virtual Private Clouds (VPC A and VPC B) on Amazon Web Services (AWS) using VPC Peering.

The architecture enforces isolation for private application workloads while providing secure administrative access to private EC2 instances from a local workstation via an SSH Bastion host using SSH ProxyJump.

📐 Architecture & Subnet CIDR Plan

AWS Region: ap-south-1 (Mumbai)

AWS Account ID: 709587652927

Key Pair: ewb-lab.pem

Resource Inventory & Addressing Scheme

Resource / Tag Name

AWS Service

VPC / Subnet

IPv4 CIDR / IP Address

Public IP Assigned

ewb-vpc-a

VPC

N/A

$10.10.0.0/16$

N/A

ewb-a-public

Subnet

ewb-vpc-a

$10.10.1.0/24$

Auto-assign Enabled

ewb-a-private

Subnet

ewb-vpc-a

$10.10.2.0/24$

Disabled

ewb-vpc-b

VPC

N/A

$10.20.0.0/16$

N/A

ewb-b-private

Subnet

ewb-vpc-b

$10.20.1.0/24$

Disabled

ewb-igw-a

Internet Gateway

ewb-vpc-a

N/A

N/A

ewb-bastion

EC2 Instance

ewb-a-public

$10.10.1.10$

Enabled (e.g. $35.154.153.93$)

ewb-app-a

EC2 Instance

ewb-a-private

$10.10.2.10$

None

ewb-app-b

EC2 Instance

ewb-b-private

$10.20.1.10$

None

ewb-a-b-peer

VPC Peering Connection

ewb-vpc-a $\leftrightarrow$ ewb-vpc-b

N/A

N/A

🔒 Security Group Configuration

1. ewb-sg-bastion (Associated with ewb-vpc-a / ewb-bastion)

Inbound Rules:

Type: SSH (Port 22) | Source: YOUR_PUBLIC_IP/32 (e.g., $106.222.220.106/32$)

2. ewb-sg-app-a (Associated with ewb-vpc-a / ewb-app-a)

Inbound Rules:

Rule 1: SSH (Port 22) | Source: $10.10.1.10/32$ (ewb-bastion private IP)

Rule 2: Custom ICMP - IPv4 (Echo Request) | Source: $10.20.1.10/32$ (ewb-app-b private IP)

3. ewb-sg-app-b (Associated with ewb-vpc-b / ewb-app-b)

Inbound Rules:

Rule 1: SSH (Port 22) | Source: $10.10.1.10/32$ (ewb-bastion private IP)

Rule 2: Custom ICMP - IPv4 (Echo Request) | Source: $10.10.2.10/32$ (ewb-app-a private IP)

🗺️ Route Tables & Routing Rules

1. ewb-rt-a-public (Associated with ewb-a-public)

Destination

Target

Purpose

$10.10.0.0/16$

local

VPC A Internal Routing

$0.0.0.0/0$

ewb-igw-a

Public Internet Access for Bastion

$10.20.0.0/16$

ewb-a-b-peer (pcx-ID)

ProxyJump SSH traffic routing to VPC B

2. ewb-rt-a-private (Associated with ewb-a-private)

Destination

Target

Purpose

$10.10.0.0/16$

local

VPC A Internal Routing

$10.20.0.0/16$

ewb-a-b-peer (pcx-ID)

Direct private traffic from App-A to VPC B

3. ewb-rt-b-private (Associated with ewb-b-private)

Destination

Target

Purpose

$10.20.0.0/16$

local

VPC B Internal Routing

$10.10.0.0/16$

ewb-a-b-peer (pcx-ID)

Return path traffic from App-B back to VPC A

💻 Local SSH Client Setup (ProxyJump)

To connect directly to the private workload instances without exposing private keys on remote servers, configure the local SSH client file (~/.ssh/config or C:\Users\<Username>\.ssh\config):

Host ewb-bastion
    HostName BASTION_PUBLIC_IP
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion


🧪 Verification & Acceptance Testing

Test 01: Baseline Pre-Peering Check

Before establishing peering routes, executing ping -c 4 10.20.1.10 from ewb-app-a results in complete packet loss, confirming strict isolated boundaries between separate VPCs.

Test 02: Forward Path Ping Test (App-A $\rightarrow$ App-B)

Connecting to ewb-app-a and sending ICMP echo requests across the active VPC peering link:

[ec2-user@ip-10-10-2-10 ~]$ ping -c 4 10.20.1.10


Output:

--- 10.20.1.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3135ms
rtt min/avg/max/mdev = 0.205/0.392/0.566/0.135 ms


Test 03: Reverse Path Ping Test (App-B $\rightarrow$ App-A)

Connecting to ewb-app-b and pinging back to ewb-app-a:

[ec2-user@ip-10-20-1-10 ~]$ ping -c 4 10.10.2.10


Output: 4 packets transmitted, 4 received, 0% packet loss.

Test 04: Private SSH Administration Access

Verified SSH access to isolated instances using single-line terminal command from local machine Command Prompt / PowerShell:

ssh -F config ewb-app-b


Result: Shell access granted on ewb-app-b without assigning a public IP address.

Test 05: Controlled Security Group Failure & Recovery

Rule Removal: Temporarily deleted the ICMP Echo Request inbound rule on ewb-sg-app-b. Re-running ping -c 4 10.20.1.10 resulted in 100% packet loss / Request Timeout.

Recovery: Re-added the ICMP inbound rule for $10.10.2.10/32$. Ping test immediately resumed with 0% packet loss, proving stateful firewall and security group enforcement.

🧹 Resource Teardown & Cleanup Order

To prevent incurring unwanted AWS charges, resources were destroyed in the following strict dependency sequence:

EC2 Instances: Terminated ewb-bastion, ewb-app-a, and ewb-app-b (releasing all mapped IP dependencies).

VPC Peering: Deleted ewb-a-b-peer connection.

Internet Gateway: Detached ewb-igw-a from ewb-vpc-a, followed by deletion.

VPCs & Subnets: Deleted ewb-vpc-a and ewb-vpc-b (which removed subnets ewb-a-public, ewb-a-private, ewb-b-private, and custom route tables).

Security Groups: Deleted ewb-sg-bastion, ewb-sg-app-a, and ewb-sg-app-b.
