# Network Security Lab Assignment - Step-by-Step Plan

## Overview
Design and setup a secure network for a small company with web server, DNS, NAT, and TLS.

**Time Allocation: 6 Hours**
- Planning & Setup: 1 hour
- Network Configuration: 2 hours  
- Services Configuration: 2.5 hours
- Testing & Documentation: 30 minutes

## Phase 1: Initial Setup & Planning (60 minutes)

### Step 1: Physical Network Setup (15 minutes)
1. Boot all 6 grml computers (Select grml 2024.02 Standard)
2. Connect computers via Ethernet switches:
   - grml1: NAT Gateway (connect to Internet)
   - grml2: DNS Server
   - grml3: Web Server
   - grml4-5: Client Workstations
   - grml6: Management/Backup

### Step 2: IP Address Planning (15 minutes)
- Public IP: Assigned by lab network
- Internal Network: 192.168.1.0/24
- Gateway: 192.168.1.1 (grml1)
- DNS: 192.168.1.2 (grml2)
- Web Server: 192.168.1.3 (grml3)
- Clients: 192.168.1.4-5 (grml4-5)
- Management: 192.168.1.6 (grml6)

### Step 3: Basic System Updates (30 minutes)
```bash
# On each machine
sudo apt update
sudo apt upgrade -y
```

## Phase 2: Network Infrastructure (120 minutes)

### Step 4: Configure NAT Gateway (grml1) (45 minutes)
1. Configure network interfaces
2. Enable IP forwarding
3. Set up iptables NAT rules
4. Configure port forwarding for web server
5. Basic firewall rules

### Step 5: Configure Internal Network (grml2-6) (45 minutes)
1. Set static IP addresses on each machine
2. Configure default gateway
3. Set DNS server (will be configured next)
4. Test internal connectivity

### Step 6: Network Connectivity Testing (30 minutes)
1. Test internal ping between all machines
2. Test external connectivity from gateway
3. Verify NAT is working

## Phase 3: Service Configuration (150 minutes)

### Step 7: DNS Server Setup (grml2) (60 minutes)
1. Install BIND9
2. Configure named.conf
3. Create forward and reverse zones
4. Configure forwarders for external DNS
5. Test DNS resolution

### Step 8: Web Server Setup (grml3) (90 minutes)
1. Install Apache2
2. Configure basic website
3. **Use one of the provided lab domains** (check with instructor)
4. Install Certbot for Let's Encrypt
5. **Configure TLS/SSL certificate (must be issued on submission date)**
6. Configure Apache for HTTPS
7. Test HTTPS access from both internal and external networks

## Phase 4: Testing & Validation (30 minutes)

### Step 9: Comprehensive Testing (20 minutes)
1. Test DNS resolution from clients
2. Test web server access from internal network
3. Test external HTTPS access
4. Verify TLS certificate validity
5. Test NAT functionality

### Step 10: Documentation & Team Preparation (10 minutes)
1. Document network configuration with detailed explanations
2. Note any issues encountered and solutions
3. **Prepare team members to explain all design components**
4. **Verify certificate was issued on submission date**
5. **Run final nmap security scan to ensure no unnecessary ports open**
6. Prepare demonstration script for presentation

## Key Security Considerations
- Firewall rules for minimal attack surface
- TLS/HTTPS for encrypted communication
- DNS security (no open recursion)
- NAT for internal network protection
- Regular security updates

## Backup Plans
- If Let's Encrypt fails: Use self-signed certificates
- If public access fails: Demonstrate local HTTPS
- If DNS fails: Use static /etc/hosts entries
- Keep grml6 as backup/troubleshooting machine

## Testing Checklist
- [ ] Internal network connectivity
- [ ] DNS resolution (internal and external)
- [ ] Web server accessible internally
- [ ] HTTPS/TLS working
- [ ] External web access (if possible)
- [ ] NAT functioning correctly
- [ ] Firewall rules active 