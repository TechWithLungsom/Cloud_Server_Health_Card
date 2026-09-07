# Server Health Card — IIS Cloud Deployment Lab

Self-reporting Windows Server 2022 virtual machine deployed on AWS EC2 serving real-time host metrics via Internet Information Services (IIS).

---

## 1. Deployment Details

These details correspond to the parameters defined in `deployment.json`:

* **Cloud Provider:** AWS
* **Region:** ap-south-1 (Asia Pacific - Mumbai)
* **Availability Zone:** ap-south-1b
* **Instance Type / Size:** t3.micro
* **Operating System:** Microsoft Windows Server 2022 Base
* **Owner:** Lungsom

---

## 2. Lab Checkpoints

| Checkpoint | Description | Screenshot |
| :--- | :--- | :--- |
| **Checkpoint 1** | Windows Server 2022 Desktop with Server Manager Open | `![Checkpoint 1](screenshots/checkpoint-1.png)` |
| **Checkpoint 2** | IIS Manager showing `HealthCard` website started on Port 80 | `![Checkpoint 2](screenshots/checkpoint-2.png)` |
| **Checkpoint 3** | Health Card dashboard loading locally with hostname and dual IPs | `![Checkpoint 3](screenshots/checkpoint-3.png)` |
| **Checkpoint 4** | Scheduled Task heartbeat strip showing 5+ collector executions | `![Checkpoint 4](screenshots/checkpoint-4.png)` |
| **Checkpoint 5** | Health Card accessed from client browser (Mac) via Public IPv4 | `![Checkpoint 5](screenshots/checkpoint-5.png)` |
| **Checkpoint 6** | Automated verification test suite showing all 9 checks PASS | `![Checkpoint 6](screenshots/checkpoint-6.png)` |

---

## 3. Written Answers

### 1. Your VM has a public IP address. Why does `ipconfig` not show it?
The EC2 virtual machine resides inside an AWS Virtual Private Cloud (VPC) subnet on a private network interface (e.g., `172.31.x.x`). The public IPv4 address belongs to AWS's public edge/gateway infrastructure, which performs 1:1 Network Address Translation (NAT). Windows only sees the local IP bound to its virtual NIC and is unaware of the NAT translation occurring outside the hypervisor.

### 2. The outbound call to `api.ipify.org` needed no firewall change, but your laptop's inbound request needed two rules. Why is the default asymmetric?
Cloud networks and host operating systems use stateful firewalls designed on a principle of least privilege:
* **Outbound traffic is allowed by default** because client requests initiate outward, and stateful tracking automatically permits return response packets.
* **Inbound traffic is blocked by default** to prevent unauthorized scanning, exploit attempts, and external attacks on listening ports. An explicit rule is required to permit unsolicited incoming connections.

### 3. Name the two firewalls you configured. What happens if you open port 80 in only one?
The two firewalls are:
1. **AWS Security Group** (cloud perimeter firewall at the hypervisor level)
2. **Windows Defender Firewall** (host-level operating system firewall)

Firewalls act in series (defense-in-depth). If port 80 is open in only one, **all inbound HTTP traffic is dropped**:
* If allowed in AWS SG but blocked in Windows: AWS forwards the packets, but Windows drops them at the OS interface.
* If allowed in Windows but blocked in AWS SG: AWS drops the packets before they reach the virtual machine.

### 4. Why does the scheduled task run as `SYSTEM` rather than as `Administrator`?
* The `NT AUTHORITY\SYSTEM` account runs in the background independently of any interactive user session. If the task ran as `Administrator`, it would stop running when the user logged off or disconnected from RDP unless credentials were saved.
* Running as `SYSTEM` does not require hardcoding or storing an administrator password that could expire or be exposed.
* It ensures the collector starts automatically on system boot (`AtStartup` trigger) before any user logs in.

### 5. After stop/start, what changed and what stayed the same — and why?
* **What changed:** The **Public IPv4 address**. Dynamic AWS public IPs are drawn from an ephemeral public pool. When an instance is stopped, AWS releases that IP back into the pool. A new IP is allocated upon startup unless an Elastic IP is reserved.
* **What stayed the same:** The **Private IP address**, **Hostname**, and **disk storage**. The private IP is tied to the Elastic Network Interface (ENI) inside the VPC subnet, which remains intact while the instance is stopped. The persistent EBS root volume retains all installed features and files.

### 6. Name one thing you had to do in your cloud's console that would have been different in the other two.
* **AWS:** Decrypting the generated Administrator password required downloading an RSA private key pair (`.pem` file) and uploading it to the AWS EC2 console to decrypt the Windows administrator password.
* **Azure / GCP:** Azure requires you to set the administrator credentials directly in the VM creation wizard (rejecting default usernames like `Administrator`). GCP creates Windows accounts on demand and generates passwords directly in the console without requiring local key decryption.