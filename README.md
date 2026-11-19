# Campus Network Security & Application Defense

A Containerlab simulation of a 4-subnet campus network with iptables firewall
and OWASP ModSecurity WAF.

## Network Topology

┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │ Student │ │ Faculty │ │ IT Admin │ │ Services Subnet │ │ 10.10.10.0/24│ │ 10.10.20.0/24│ │ 10.10.30.0/24│ │ 10.10.40.0/24 │ │ │ │ │ │ │ │ │ │ student-cli │ │ faculty-cli │ │ admin-cli │ │ WAF 10.10.40.80:8080 │ │ │ │ │ │ │ │ JuiceShop 10.10.40.81 │ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ │ PGDB 10.10.40.100 │ │ │ │ └──────────┬───────────────┘ └──────────────────┴──────────────────┘ │ │ │ ┌──────┴──────┐ ┌───────┴────────┐ │ ROUTER │ │ SWITCH │ │ 10.10.10.1 │ │ 10.10.40.1 │ │ 10.10.20.1 │ │ (sw-services) │ │ 10.10.30.1 │ └────────────────┘ │ 10.10.40.1 │ │ iptables │ │ DROP │ └─────────────┘


## Security Policies

| Policy | Description |
|--------|-------------|
| Gateway | Stateful forwarding with default DROP on the FORWARD chain |
| SSH | Only IT Admin subnet (10.10.30.0/24) may reach port 22 |
| Web App | All clients must access Juice Shop via WAF (10.10.40.80:8080) |
| Database | PostgreSQL (10.10.40.100:5432) accessible only from Faculty & IT Admin |
| WAF | ModSecurity blocks SQL injection attempts with 403 Forbidden |

## Prerequisites

- **Docker** (v20.10+)
- **Containerlab** (v0.60+)
- **Linux host** (required for `kind: linux` nodes)

## Build Local Images

The base images don't include `iproute2` (needed for static IP config). Build them first:

```bash
# Build Juice Shop image (with networking utilities)
docker build -t custom-juiceshop:latest ./images/juiceshop

# Build ModSecurity WAF image (with networking utilities)
docker build -t custom-waf:latest ./images/modsec

# Container deployment
sudo clab deploy -t lab.clab.yml

# Destroy Container
sudo clab destroy -t lab.clab.yml

# Verification and automated test suite
Your topology must successfully run and pass all six of the following test cases executed directly via docker exec:

Test 1: Legitimate Bulletin Web Access (Student to WAF)
Command: docker exec -it clab-campus-network-security-student-client curl -s -o /dev/null -w "%{http_code}" http://10.10.40.80:8080/
Expected Result: 200

Test 2: ModSecurity WAF Interception of SQL Injection
Command: docker exec -it clab-campus-network-security-student-client curl -s -o /dev/null -w "%{http_code}" "http://10.10.40.80:8080/rest/user/login?email='OR'1'='1"
Expected Result: 403

Test 3: Authorized Grading DB Access (Faculty Subnet)
Command: docker exec -it clab-campus-network-security-faculty-client nc -zv -w 3 10.10.40.100 5432
Expected Result: open / connection succeeded

Test 4: Unauthorized Grading DB Access Block (Student Subnet)
Command: docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.100 5432
Expected Result: Connection timeout / failure

Test 5: SSH Privilege Validation (IT Admin Host Allowed)
Command: docker exec -it clab-campus-network-security-admin-client nc -zv -w 3 10.10.40.100 22
Expected Result: Connection refused (packet reaches target, confirming rule allows traversal, but port 22 is unlistened on DB)

Test 6: SSH Privilege Restriction (Student Host Blocked)
Command: docker exec -it clab-campus-network-security-student-client nc -zv -w 3 10.10.40.100 22
Expected Result: Connection timeout / failure (dropped by router firewall)
