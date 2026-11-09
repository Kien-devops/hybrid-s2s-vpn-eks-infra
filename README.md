# Hybrid S2S VPN + EKS Platform

Nền tảng hybrid tập trung vào **AWS Site-to-Site VPN, DPD, BGP và routing hai chiều** giữa một Ubuntu server đóng vai trò on-premises Customer Gateway và một Amazon EKS private cluster trên AWS.

Repository này chỉ mô tả và quản lý **hạ tầng, network design, runbook, verification và cleanup**. Mã nguồn ứng dụng nằm tại [hybrid-s2s-vpn-eks-app](https://github.com/Kien-devops/hybrid-s2s-vpn-eks-app).

> Trạng thái hiện tại: architecture baseline. Chưa tuyên bố đã deploy hoặc đã có runtime evidence.

## 1. Mục tiêu

- Kết nối Ubuntu on-premises với AWS bằng AWS Site-to-Site VPN.
- Sử dụng hai IPsec tunnels do AWS cung cấp.
- Dùng IKEv2 và DPD để phát hiện peer/tunnel không còn hoạt động.
- Dùng dynamic routing bằng BGP thay vì quản lý static route thủ công.
- Cho phép Ubuntu gọi private application trên EKS qua internal Network Load Balancer.
- Cho phép EKS `order-api` gọi PostgreSQL nằm trên Ubuntu qua VPN.
- Đo và ghi lại route convergence, tunnel failover, packet path và response path.
- Giữ scope nhỏ: network hybrid là trọng tâm; application chỉ tạo traffic thật để kiểm chứng.

## 2. Phạm vi và giới hạn

### Trong phạm vi

- Một Ubuntu server làm Customer Gateway, FRR/BGP router và on-premises workload host.
- Một VPC tại `ap-southeast-2`.
- Một Virtual Private Gateway.
- Một AWS Site-to-Site VPN connection với hai tunnels.
- Một private EKS cluster trải trên hai Availability Zones.
- Một internal NLB.
- Hai service trên EKS: `frontend` và `order-api`.
- Một PostgreSQL instance trên Ubuntu.
- DPD, BGP, route propagation, security groups, MTU/MSS và packet capture.

### Ngoài phạm vi MVP

- AWS Direct Connect.
- Transit Gateway.
- Gateway Load Balancer, AWS Network Firewall hoặc Suricata.
- Multi-Region DR.
- Service mesh.
- Public database endpoint.
- Expose PostgreSQL trực tiếp ra Internet.
- Dual physical Customer Gateway.

> Hai VPN tunnels tạo redundancy cho tunnel/IPsec path. Một Ubuntu server vẫn là single point of failure: nếu Ubuntu chết, cả hai tunnels và PostgreSQL cùng mất.

## 3. Kiến trúc tổng thể

```text
┌──────────────────────────── ON-PREMISES / LAB ────────────────────────────┐
│ Ubuntu Server                                                               │
│ Local CIDR: 10.200.0.0/16                                                  │
│                                                                            │
│  strongSwan                                                               │
│  ├── IPsec Tunnel 1 ───────────────────────────────────────────────┐       │
│  └── IPsec Tunnel 2 ────────────────────────────────────────────┐  │       │
│                                                                  │  │       │
│  FRRouting / FRR                                                   │  │       │
│  ├── BGP ASN 65010                                                │  │       │
│  ├── Advertise 10.200.0.0/16                                    │  │       │
│  └── Learn 10.100.0.0/16                                        │  │       │
│                                                                  │  │       │
│  PostgreSQL: 10.200.10.10:5432                                  │  │       │
│  Test client: curl, ping, iperf3, tcpdump                       │  │       │
└──────────────────────────────────────────────────────────────────┼──┼───────┘
                                                                   │  │
                              Internet / UDP 500 / UDP 4500        │  │
                                                                   ▼  ▼
┌──────────────────────────────────── AWS VPC ────────────────────────────────┐
│ VPC CIDR: 10.100.0.0/16                                                   │
│ Region: ap-southeast-2                                                     │
│                                                                            │
│  Customer Gateway → Site-to-Site VPN → Virtual Private Gateway             │
│                            IKEv2 + DPD + dynamic BGP                       │
│                                      │                                     │
│                                      ▼                                     │
│  VPC route tables: 10.200.0.0/16 → VGW propagated route                   │
│                                      │                                     │
│  ┌──────────────────────────── Amazon EKS ──────────────────────────────┐  │
│  │ Private subnets in two AZs                                           │  │
│  │                                                                      │  │
│  │  Internal NLB: app.hybrid.internal                                  │  │
│  │       │                                                              │  │
│  │       ▼                                                              │  │
│  │  frontend Service → frontend Pods                                   │  │
│  │       │ ClusterIP                                                    │  │
│  │       ▼                                                              │  │
│  │  order-api Service → order-api Pods                                 │  │
│  │       │                                                              │  │
│  │       │ TCP 5432 through VPN                                         │  │
│  │       ▼                                                              │  │
│  │  PostgreSQL on Ubuntu: 10.200.10.10:5432                            │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

## 4. Addressing plan

| Domain | CIDR / address | Purpose |
|---|---|---|
| AWS VPC | `10.100.0.0/16` | VPC CIDR; must not overlap on-premises |
| EKS private subnet AZ-a | `10.100.10.0/24` | Nodes and VPC CNI pod IPs |
| EKS private subnet AZ-b | `10.100.20.0/24` | Nodes and VPC CNI pod IPs |
| On-premises network | `10.200.0.0/16` | Ubuntu-side advertised CIDR |
| PostgreSQL | `10.200.10.10:5432` | On-premises database endpoint |
| On-premises test client | `10.200.10.20` | curl, ping, iperf3 and packet tests |
| On-premises BGP ASN | `65010` | FRR ASN; use a different ASN from AWS |

The final subnet sizes and Pod IP allocation must be validated against the selected EKS CNI configuration before deployment.

## 5. Traffic flows

### 5.1 On-premises to EKS

```text
Ubuntu test client
  → app.hybrid.internal
  → internal NLB private IP
  → VPC route / VPN path
  → AWS VGW
  → EKS frontend Service
  → frontend Pod
  → order-api Service
  → order-api Pod
  → HTTP response through the same hybrid path
```

### 5.2 EKS to on-premises PostgreSQL

```text
order-api Pod
  → destination 10.200.10.10:5432
  → EKS subnet route table
  → propagated route 10.200.0.0/16 via VGW
  → selected VPN tunnel
  → Ubuntu PostgreSQL
  → return route 10.100.0.0/16 via FRR/IPsec
  → order-api Pod
```

### 5.3 Application services

`frontend` is the only service exposed through the internal NLB. It calls `order-api` through a Kubernetes `ClusterIP` Service.

`order-api` owns the database connection and calls PostgreSQL on Ubuntu. It must return health and dependency status without exposing credentials or sensitive rows.

Suggested endpoints:

```text
GET  /health
GET  /orders
POST /orders
GET  /dependency-check
```

The application is a verification workload, not the project focus. A successful request must prove the complete chain:

```text
Ubuntu → VPN → VGW → VPC routing → internal NLB → EKS frontend
→ EKS order-api → VPN → on-premises PostgreSQL → response
```

## 6. Routing design

### 6.1 Ubuntu / FRR

```text
Advertise:
  10.200.0.0/16

Learn from AWS:
  10.100.0.0/16

Expected forwarding:
  10.100.0.0/16 → IPsec/XFRM policy
```

### 6.2 AWS route tables

```text
Destination       Target
10.100.0.0/16     local
10.200.0.0/16     VGW propagated route
0.0.0.0/0         NAT Gateway only if private nodes require outbound access
```

Enable route propagation only on the route tables associated with the EKS subnets and the subnets that host the internal NLB path. Verify the actual propagated route after BGP establishment; do not treat a configured route as proof of working data-plane traffic.

### 6.3 Routing invariants

- AWS VPC CIDR and on-premises CIDR never overlap.
- Both directions have a route for the remote CIDR.
- PostgreSQL return traffic uses the VPN path, not a default Internet route.
- Security Groups allow only the required source CIDRs and ports.
- NACLs do not block ephemeral response ports.
- The selected EKS networking mode must provide reachable Pod/Service targets from the intended path.

## 7. VPN, DPD and BGP behavior

```text
Tunnel 1 and Tunnel 2 establish IKEv2/IPsec
  ↓
DPD sends liveness checks
  ↓
FRR maintains one BGP session per tunnel
  ↓
AWS and Ubuntu exchange remote CIDRs
  ↓
VGW propagates 10.200.0.0/16 into selected VPC route tables
```

Failure sequence:

```text
Tunnel 1 stops responding
  → DPD timeout
  → AWS/Ubuntu clear or restart the IKE session according to tunnel policy
  → BGP peer 1 goes down
  → route path through peer 1 is withdrawn
  → peer 2 remains established
  → traffic converges through Tunnel 2
```

DPD detects peer/IPsec liveness. BGP controls route advertisement and withdrawal. Neither mechanism alone proves that the application is healthy; application and packet tests remain required.

## 8. Verification plan

### Network control plane

- Both VPN tunnels show the expected state.
- DPD/IKE events are visible in VPN logs.
- Both BGP sessions reach `Established`.
- Ubuntu advertises `10.200.0.0/16`.
- Ubuntu learns `10.100.0.0/16`.
- AWS route tables show the propagated on-premises route.

### Network data plane

```bash
ping <private-eks-or-nlb-address>
curl -v http://app.hybrid.internal/health
traceroute <private-address>
tcpdump -ni any host 10.100.0.0/16 or host 10.200.0.0/16
```

Use the actual address assigned by the deployment. Do not use Internet-facing endpoints for the primary test.

### Application path

```text
1. Ubuntu → internal NLB /health
2. Ubuntu → internal NLB /dependency-check
3. frontend → order-api
4. order-api → PostgreSQL on Ubuntu
5. POST /orders and GET /orders
6. Verify response, logs, correlation ID and database result
```

### Failover test

Run a continuous request loop from Ubuntu while disabling only Tunnel 1:

```bash
while true; do
  date -Is
  curl --connect-timeout 2 --max-time 5 -fsS \
    http://app.hybrid.internal/dependency-check
  sleep 1
done
```

Record:

- Last successful request before failure.
- DPD timeout timestamp.
- BGP peer-down timestamp.
- Route withdrawal timestamp.
- First successful request through Tunnel 2.
- Packet loss and recovery duration.
- Any duplicate order or failed database transaction.

## 9. Failure scenarios

| Scenario | Expected observation |
|---|---|
| Stop Tunnel 1 | DPD/IKE event, BGP peer 1 down, traffic uses Tunnel 2 |
| Stop Tunnel 2 | Tunnel 1 carries traffic |
| Stop FRR | BGP routes withdraw; IPsec may remain up |
| Block UDP 500/4500 | IKE/DPD failure; tunnel cannot re-establish |
| Remove AWS route propagation | Tunnel may be up but data plane fails |
| Block TCP 5432 | EKS health endpoint shows database dependency failure |
| Break PostgreSQL return route | SYN reaches DB but response fails; packet capture shows asymmetry |
| Inject MTU/MSS issue | Large requests fail or fragment; small health checks may pass |

Every test must have a stop condition and a restoration step. Do not run destructive tests against production infrastructure.

## 10. Evidence package

Store redacted evidence outside source secrets:

```text
evidence/
├── topology/
├── vpn/
│   ├── tunnel-state.txt
│   ├── ike-dpd-events.txt
│   └── bgp-summary.txt
├── routing/
│   ├── aws-route-tables.txt
│   ├── ubuntu-routes.txt
│   └── route-convergence.csv
├── traffic/
│   ├── request-response.txt
│   └── packet-capture-summary.txt
├── application/
│   ├── frontend-health.json
│   ├── order-api-health.json
│   └── database-dependency-check.json
└── reports/
    └── failover-report.md
```

Never commit PSKs, private keys, database passwords, AWS credentials, full tokens or unredacted account identifiers.

## 11. Implementation order

1. Confirm Ubuntu public reachability, kernel forwarding and non-overlapping CIDRs.
2. Create the AWS VPC and private subnets in two AZs.
3. Create the Customer Gateway with the Ubuntu public address.
4. Create and attach the Virtual Private Gateway.
5. Create the Site-to-Site VPN with IKEv2, DPD and dynamic BGP.
6. Configure strongSwan from the AWS-generated tunnel parameters.
7. Configure FRR with the AWS BGP parameters and advertise the on-premises CIDR.
8. Verify both tunnels, DPD, BGP and propagated routes before deploying EKS workloads.
9. Create the private EKS cluster and deploy `frontend` and `order-api`.
10. Expose only `frontend` through an internal NLB.
11. Configure PostgreSQL access from `order-api` over the VPN.
12. Execute bidirectional traffic, failure injection and failover tests.
13. Save redacted evidence and complete cleanup.

## 12. Cost and cleanup

Potential hourly or usage-based cost comes from EKS control plane, worker capacity, internal NLB, NAT Gateway, VPN connection, public IPv4, PostgreSQL storage/compute and CloudWatch logs.

Cleanup order:

```text
1. Stop test traffic and application writes
2. Delete Kubernetes Ingress/LoadBalancer resources
3. Delete EKS Services, Deployments and cluster add-ons
4. Delete EKS node groups and cluster
5. Delete route propagation changes and VPC attachments
6. Delete Site-to-Site VPN
7. Delete Virtual Private Gateway
8. Delete Customer Gateway if no longer needed
9. Delete NAT Gateway and release Elastic IPs
10. Delete subnets, route tables, security groups and VPC
11. Stop/remove Ubuntu PostgreSQL and local test artifacts
12. Verify AWS Cost Explorer, resource inventory and CloudWatch retention
```

Do not delete a shared Customer Gateway, VPC, EIP or logging resource without confirming it belongs exclusively to this project.

## 13. Acceptance criteria

The project is complete only when all criteria have evidence:

- Ubuntu calls the EKS internal NLB over the VPN and receives HTTP `200`.
- `frontend` calls `order-api` through a private Kubernetes Service.
- `order-api` reaches PostgreSQL on Ubuntu through the VPN.
- Return traffic succeeds without asymmetric routing.
- Both BGP sessions establish and exchange the expected CIDRs.
- DPD detects a failed tunnel.
- Tunnel 2 carries traffic after Tunnel 1 failure.
- Recovery time, packet loss and route convergence are recorded.
- At least one route failure and one MTU/MSS failure are diagnosed from evidence.
- No secret or private key is committed.
- Cleanup removes billable lab resources or documents an explicit retained resource.

## Related repository

- Application source: [Kien-devops/hybrid-s2s-vpn-eks-app](https://github.com/Kien-devops/hybrid-s2s-vpn-eks-app)
