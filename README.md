# EWB AWS VPC Peering Lab

## Project overview
This project connects two non-overlapping AWS VPCs using VPC Peering. A public bastion host in VPC A provides administrative access to two private EC2 workloads. The application instances communicate using private IPv4 addresses.

**Region:** Asia Pacific (Sydney) — `ap-southeast-2`  
**OS:** Amazon Linux 2023  
**Key pair:** `ewb-lab`

## Architecture and addressing

| Resource | Name | CIDR / Private IPv4 |
|---|---|---|
| VPC A | `ewb-vpc-a` | `10.10.0.0/16` |
| Public subnet A | `ewb-a-public` | `10.10.1.0/24` |
| Private subnet A | `ewb-a-private` | `10.10.2.0/24` |
| VPC B | `ewb-vpc-b` | `10.20.0.0/16` |
| Private subnet B | `ewb-b-private` | `10.20.1.0/24` |
| Bastion | `ewb-bastion` | `10.10.1.233` |
| App-A | `ewb-app-a` | `10.10.2.184` |
| App-B | `ewb-app-b` | `10.20.1.31` |
| Peering | `ewb-a-b-peer` | Active (confirmed in console) |

The two VPC CIDR ranges do not overlap. App-A and App-B are private workloads and should have no public IPv4 addresses. The bastion is the only intended internet-facing instance.

## Route tables

| Route table | Destination | Target |
|---|---|---|
| `ewb-rt-a-public` | `10.10.0.0/16` | local |
| `ewb-rt-a-public` | `0.0.0.0/0` | `ewb-igw-a` |
| `ewb-rt-a-public` | `10.20.0.0/16` | `ewb-a-b-peer` |
| `ewb-rt-a-private` | `10.10.0.0/16` | local |
| `ewb-rt-a-private` | `10.20.0.0/16` | `ewb-a-b-peer` |
| `ewb-rt-b-private` | `10.20.0.0/16` | local |
| `ewb-rt-b-private` | `10.10.0.0/16` | `ewb-a-b-peer` |

The peering route is required on both sides so traffic and return traffic can flow. The public subnet route table also needs the VPC B route because the bastion initiates SSH connections to private instances in VPC B.

## Security groups

| Security group | Inbound access |
|---|---|
| `ewb-sg-bastion` | SSH TCP 22 from the administrator's current public IP `/32` |
| `ewb-sg-app-a` | SSH TCP 22 from `10.10.1.233/32`; ICMP from `10.20.1.31/32` (and the configured bastion ICMP test source if retained) |
| `ewb-sg-app-b` | SSH TCP 22 from `10.10.1.233/32`; ICMP from `10.10.2.184/32` (and the configured bastion ICMP test source if retained) |

Keep SSH restricted to the required source addresses. Do not use `0.0.0.0/0` for SSH. Default outbound rules were left available for the lab tests.

## Connectivity validation performed

| Test | Result recorded |
|---|---|
| VPC peering state | Active |
| Windows laptop → Bastion SSH | Successful |
| Bastion → App-A ICMP ping | Successful, reported 0% packet loss |
| Bastion → App-B ICMP ping | Successful, reported 0% packet loss |
| Bastion → App-A TCP 22 | Successful; Python socket `connect_ex()` returned `0` |
| Laptop → App-A through Bastion (`ssh ewb-app-a`) | Successful |
| App-A → App-B ICMP ping | Successful |

**Important:** These are the tests actually confirmed during the lab conversation. Do not claim the following until you run them and save screenshots: App-B → App-A reverse ping; laptop SSH through Bastion to App-B; baseline ping failure before peering; controlled ICMP rule-removal failure and recovery; and final cleanup.

## Commands used

From Bastion to App-A:
```bash
ping -c 4 10.10.2.184
```

From Bastion to App-B:
```bash
ping -c 4 10.20.1.31
```

Check TCP SSH port from Bastion (Python alternative when `nc` is unavailable):
```bash
python3 -c "import socket; s=socket.socket(); s.settimeout(5); print(s.connect_ex(('10.10.2.184',22))); s.close()"
```
A result of `0` means the TCP connection to port 22 succeeded.

From App-A to App-B:
```bash
ping -c 4 10.20.1.31
```

Windows SSH configuration used a local key and ProxyJump:
```sshconfig
Host ewb-bastion
    HostName <CURRENT_BASTION_PUBLIC_IP>
    User ec2-user
    IdentityFile C:/Users/<WINDOWS_USER>/Downloads/ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.184
    User ec2-user
    IdentityFile C:/Users/<WINDOWS_USER>/Downloads/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```
Connect with:
```powershell
ssh ewb-app-a
```
Replace the placeholder Windows username and always verify the Bastion public IP in EC2 before use; public IPv4 addresses can change after relaunch. Never commit the `.pem` file or private key contents to GitHub.

## Screenshots to add before final submission

Create an `evidence/` folder and add clearly named screenshots:
1. `01-vpc-and-subnets.png` — both VPCs and all subnet CIDRs.
2. `02-peering-active.png` — `ewb-a-b-peer` in Active state.
3. `03-route-tables.png` — all three route tables and subnet associations.
4. `04-security-groups.png` — inbound rules for bastion, App-A and App-B.
5. `05-ec2-instances.png` — all three instances; show that App-A and App-B have no public IPv4.
6. `06-bastion-to-app-a-ping.png`
7. `07-bastion-to-app-b-ping.png`
8. `08-app-a-to-app-b-ping.png`
9. `09-app-b-to-app-a-ping.png` — **still to be captured**.
10. `10-private-ssh-to-app-b.png` — **still to be captured**.
11. `11-baseline-before-peering.png` — **only if genuinely performed before peering**.
12. `12-controlled-failure-and-recovery.png` — remove only the App-B inbound ICMP rule from App-A, demonstrate failure, restore it and demonstrate recovery.
13. `13-cleanup.png` — cleanup evidence after saving all project evidence.

## Learning outcomes
- Designed non-overlapping VPC and subnet CIDR ranges.
- Used public and private subnets with explicit route-table associations.
- Used an Internet Gateway for the bastion's internet reachability.
- Established VPC Peering and configured forward and return routes.
- Restricted inbound SSH and ICMP using security-group source addresses.
- Used a bastion host and local SSH ProxyJump configuration to administer private instances.
- Verified private IP connectivity across the peering connection.

## Cleanup and cost control
After capturing evidence, follow the workbook cleanup sequence. Terminate the three EC2 instances; verify and remove leftover lab EBS volumes and snapshots; release any Elastic IP created; delete the peering connection and lab security groups after network interfaces are gone; remove lab subnets and custom route tables; detach/delete the lab Internet Gateway; then delete both lab VPCs. Check the Sydney region for remaining billable resources. Do not delete resources until the required screenshots and report are saved.

## Resource IDs to fill in
Record actual AWS IDs before final submission:
- VPC A ID: `TODO`
- VPC B ID: `TODO`
- Public subnet A ID: `TODO`
- Private subnet A ID: `TODO`
- Private subnet B ID: `TODO`
- Peering connection ID: `TODO`
- Internet Gateway ID: `TODO`
- Route table IDs: `TODO`
- Security group IDs: `TODO`
- EC2 instance IDs: `TODO`
