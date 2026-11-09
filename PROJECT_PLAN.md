# Project Plan — Hybrid S2S VPN + EKS

## 0. Handoff Context

Cập nhật: 2026-11-09.

Đây là plan bàn giao cho agent tiếp theo khi chuyển workspace. Đọc file này trước khi sửa hoặc triển khai.

Hai repository đã được tạo:

- Infrastructure: `Kien-devops/hybrid-s2s-vpn-eks-infra`
- Application: `Kien-devops/hybrid-s2s-vpn-eks-app`

Trạng thái hiện tại:

- Infrastructure repo chỉ có `README.md` và file plan này.
- Application repo đang để trống.
- Chưa deploy AWS resource.
- Chưa cấu hình Ubuntu, strongSwan, FRR hoặc PostgreSQL.
- Chưa có runtime evidence.

## 1. Mục tiêu duy nhất

Xây dựng một lab hybrid trong đó **Ubuntu on-premises gọi được ứng dụng trên private Amazon EKS qua AWS Site-to-Site VPN**, đồng thời `order-api` trên EKS truy cập PostgreSQL chạy trên Ubuntu qua cùng kết nối hybrid.

Trọng tâm không phải viết application phức tạp. Trọng tâm là:

```text
IPsec → DPD → BGP → route propagation → VPC routing
→ internal NLB → EKS Service → Pod
→ route ngược → PostgreSQL on-premises
```

## 2. Kiến trúc đã chốt

```text
Ubuntu Server / On-premises
├── strongSwan
│   ├── AWS VPN Tunnel 1
│   ├── AWS VPN Tunnel 2
│   └── IKEv2 + DPD
├── FRRouting / FRR
│   ├── Local ASN: 65010
│   ├── Advertise: 10.200.0.0/16
│   └── Learn: 10.100.0.0/16
├── PostgreSQL: 10.200.10.10:5432
└── Test client: curl, ping, iperf3, tcpdump

AWS ap-southeast-2
├── VPC: 10.100.0.0/16
├── Private subnets in two AZs
├── Virtual Private Gateway
├── AWS Site-to-Site VPN
│   ├── Tunnel 1
│   └── Tunnel 2
└── Private EKS cluster
    ├── frontend Service/Pods
    ├── order-api Service/Pods
    └── Internal NLB → frontend only
```

## 3. Application behavior

Application repo sẽ có hai service tối giản:

### `frontend`

- Exposed through internal NLB.
- Calls `order-api` through Kubernetes `ClusterIP` Service.
- Endpoints: `/health`, `/orders`, `/dependency-check`.

### `order-api`

- Not exposed directly through NLB.
- Connects to PostgreSQL on Ubuntu at `10.200.10.10:5432`.
- Returns dependency status without returning secrets.
- Supports a minimal order read/write flow and an idempotency key for retry tests.

Primary test:

```text
Ubuntu curl
→ internal NLB
→ frontend Pod
→ order-api Pod
→ VPN
→ PostgreSQL on Ubuntu
→ response to Ubuntu
```

## 4. Scope guardrails

### Must include

- IKEv2.
- DPD.
- Two AWS VPN tunnels.
- Dynamic BGP through FRR.
- VGW route propagation.
- Bidirectional route validation.
- Private EKS in two AZs.
- Internal NLB.
- PostgreSQL on Ubuntu.
- Tunnel failover and evidence collection.

### Do not add in MVP

- Transit Gateway.
- Direct Connect.
- GWLB, Suricata or Network Firewall.
- Multi-Region DR.
- Service mesh.
- Public database access.
- Extra microservices.
- Unrequested AWS deployment.

Add advanced components only after MVP acceptance criteria pass and the user explicitly approves scope expansion.

## 5. Implementation phases

### Phase 0 — Confirm prerequisites

- Confirm Ubuntu has stable public IPv4 or a supported NAT-T path.
- Confirm no CGNAT prevents AWS from establishing VPN tunnels.
- Confirm Ubuntu can run strongSwan, FRR, PostgreSQL and packet tools.
- Confirm AWS region, account, quotas and budget.
- Confirm CIDRs do not overlap:
  - AWS: `10.100.0.0/16`
  - On-premises: `10.200.0.0/16`

Deliverable: prerequisite checklist; no AWS mutation yet.

### Phase 1 — AWS network foundation

- Create VPC and private subnets in two AZs.
- Create route tables and security groups.
- Create and attach VGW.
- Create Customer Gateway using Ubuntu public address and ASN `65010`.
- Create Site-to-Site VPN with dynamic routing, IKEv2 and DPD.

Deliverable: AWS-generated tunnel configuration saved securely, never committed with PSKs/private keys.

### Phase 2 — Ubuntu VPN and BGP

- Configure strongSwan using AWS-generated tunnel parameters.
- Configure FRR with two BGP neighbors.
- Advertise `10.200.0.0/16`.
- Receive `10.100.0.0/16`.
- Enable forwarding and firewall rules safely.

Deliverable: both IPsec tunnels up; both BGP sessions `Established`.

### Phase 3 — Route and data-plane validation

- Enable VPC route propagation on EKS subnet route tables.
- Verify AWS learned `10.200.0.0/16`.
- Verify Ubuntu learned `10.100.0.0/16`.
- Test packet path in both directions.
- Test small and large payloads.
- Record MTU/MSS behavior.

Deliverable: route matrix, `ip route`, FRR BGP output, AWS route evidence, packet capture summary.

### Phase 4 — EKS platform

- Create private EKS cluster in two AZs.
- Use Amazon VPC CNI with Pod IP allocation validated against subnet sizing.
- Configure private access required for operations.
- Deploy only the required add-ons and node capacity.
- Avoid public application exposure.

Deliverable: healthy EKS cluster and private node connectivity.

### Phase 5 — Two-service application

- Build and deploy `frontend` and `order-api` from the app repository.
- Expose only `frontend` through an internal NLB.
- Configure `frontend → order-api` using ClusterIP.
- Configure `order-api → 10.200.10.10:5432`.
- Store database credentials outside Git.

Deliverable: Ubuntu calls `/health` and `/dependency-check` with HTTP `200`.

### Phase 6 — Failure and failover tests

Run controlled tests only:

1. Disable Tunnel 1.
2. Observe DPD/IKE events.
3. Observe BGP peer 1 down.
4. Observe route withdrawal/convergence.
5. Confirm Tunnel 2 carries traffic.
6. Restore Tunnel 1.
7. Stop FRR and distinguish BGP failure from IPsec failure.
8. Test blocked UDP `500/4500`.
9. Test broken PostgreSQL return route.
10. Test MTU/MSS behavior.

Deliverable: timestamped failover report with packet loss, convergence time and application result.

### Phase 7 — Cleanup and review

- Stop application traffic.
- Remove Kubernetes LoadBalancer resources.
- Delete EKS workloads, node groups and cluster.
- Delete VPN, VGW, Customer Gateway and VPC resources in dependency order.
- Stop/remove local PostgreSQL test resources.
- Verify no billable resources remain unless explicitly retained.

Deliverable: cleanup evidence and final architecture review.

## 6. Acceptance criteria

Do not mark the project complete until all items have evidence:

- Ubuntu reaches the internal NLB over VPN.
- `frontend` reaches `order-api` through ClusterIP.
- `order-api` reaches PostgreSQL on Ubuntu through VPN.
- Response traffic returns successfully.
- Both BGP sessions establish.
- AWS learns `10.200.0.0/16`.
- Ubuntu learns `10.100.0.0/16`.
- DPD detects a failed tunnel.
- Tunnel 2 carries traffic after Tunnel 1 failure.
- Recovery time and packet loss are recorded.
- At least one route failure and one MTU/MSS issue are diagnosed.
- No credentials, PSKs or private keys enter Git.
- Cleanup status is documented.

## 7. Evidence files expected later

```text
evidence/
├── vpn/tunnel-state.txt
├── vpn/ike-dpd-events.txt
├── vpn/bgp-summary.txt
├── routing/aws-route-tables.txt
├── routing/ubuntu-routes.txt
├── routing/route-convergence.csv
├── traffic/request-response.txt
├── traffic/packet-capture-summary.txt
├── application/dependency-check.json
└── reports/failover-report.md
```

Evidence must be redacted. Do not commit secrets, private keys, PSKs, database passwords, access keys or unredacted sensitive identifiers.

## 8. Operating rules for the next agent

- Read `README.md` and this file before changing scope.
- Inspect repository state before editing.
- Keep infrastructure and application repositories separate.
- Do not create application code in the infrastructure repository.
- Do not deploy, apply, destroy or change AWS resources without explicit user instruction for that step.
- Use AWS MCP for AWS operations when available; do not replace it with ad-hoc `boto3` or shell calls.
- Treat all AWS changes as production-sensitive until the user confirms this is a lab execution.
- Preserve existing files and unrelated changes.
- Report static design evidence separately from live AWS evidence.
- Stop after the requested phase; do not auto-chain to the next phase.

## 9. First action after workspace switch

The next agent should:

1. Read this plan and `README.md`.
2. Inspect the app and infrastructure repository states.
3. Ask or confirm whether the user wants **Phase 0 planning only** or **Phase 1 AWS deployment**.
4. If Phase 0 only, produce a prerequisite checklist and final CIDR/ASN/region decision.
5. If deployment is explicitly approved, execute only the selected phase and record evidence.

## 10. Current decision log

- AWS workload: private EKS, not ECS.
- On-premises workload: PostgreSQL on the single Ubuntu server.
- Application shape: two EKS services, `frontend` and `order-api`.
- AWS entry point: internal NLB.
- VPN: AWS Site-to-Site VPN with two tunnels.
- Tunnel protocol: IKEv2/IPsec.
- Liveness: DPD.
- Routing: dynamic BGP through FRR.
- AWS routing hub: VGW for MVP.
- AWS region: `ap-southeast-2`.
- AWS VPC CIDR: `10.100.0.0/16`.
- On-premises CIDR: `10.200.0.0/16`.
- On-premises ASN: `65010`.
- No live deployment has been performed.
