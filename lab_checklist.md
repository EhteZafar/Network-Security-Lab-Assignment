# Network Security Lab - Quick Reference Checklist

## Pre-Lab Setup ✓
- [ ] Boot all 5 grml computers (grml 2024.02 Standard)
- [ ] Connect network cables and switches (need 3 switch ports for grml1)
- [ ] Verify physical connectivity
- [ ] **Updated Architecture**: DMZ implementation (Internal: 192.168.1.0/24, DMZ: 192.168.2.0/24)
- [ ] Use provided public IP: 141.76.46.220 or 141.76.46.221

## Phase 1: Network Infrastructure (120 min)

### grml1 - NAT Gateway ✓ (Three-Zone DMZ)
- [ ] Configure network interfaces (eth0=external, eth1=DMZ, eth2=internal)
- [ ] Set IPs: External 141.76.46.220/24, DMZ 192.168.2.1/24, Internal 192.168.1.1/24
- [ ] Enable IP forwarding (`echo 1 > /proc/sys/net/ipv4/ip_forward`)
- [ ] Configure iptables NAT rules
- [ ] Set up port forwarding (80→192.168.2.10:80, 443→192.168.2.10:443)
- [ ] Configure firewall rules (default DROP, allow established)
- [ ] Test external connectivity

### grml2 - DNS Server ✓
- [ ] Set IP: 192.168.1.2/24
- [ ] Install BIND9 (`apt install bind9 bind9utils`)
- [ ] Configure `/etc/bind/named.conf.options`
- [ ] Create local zone `company.local`
- [ ] Create forward zone file `/etc/bind/db.company.local`
- [ ] Create reverse zone file `/etc/bind/db.192.168.1`
- [ ] Start BIND9 service
- [ ] Test DNS resolution locally

### grml3 - Web Server ✓ (DMZ)
- [ ] Set IP: 192.168.2.10/24 (DMZ Network)
- [ ] Install Apache2 (`apt install apache2`)
- [ ] Install Certbot (`apt install certbot python3-certbot-apache`)
- [ ] Create basic HTML website
- [ ] Configure virtual host for `www.company.local`
- [ ] Generate SSL certificate (self-signed or Let's Encrypt)
- [ ] Enable SSL/TLS (port 443)
- [ ] Configure HTTP→HTTPS redirect
- [ ] Add security headers (HSTS, etc.)

### grml4 & grml5 - Client Workstations ✓
- [ ] Set IPs: 192.168.1.4/24 and 192.168.1.5/24
- [ ] Configure DNS: 192.168.1.2
- [ ] Install browser (`apt install firefox-esr`)
- [ ] Install testing tools (`curl`, `nslookup`, `dig`)



## Phase 2: Service Configuration (120 min)

### DNS Configuration Verification ✓
- [ ] Test forward resolution: `nslookup www.company.local`
- [ ] Test reverse resolution: `nslookup 192.168.1.3`
- [ ] Test external resolution: `nslookup google.com`
- [ ] Verify from all client machines

### Web Server Configuration Verification ✓
- [ ] Test HTTP access (should redirect to HTTPS)
- [ ] Test HTTPS access with valid certificate
- [ ] Verify security headers
- [ ] Test from internal network
- [ ] Test from external network (if possible)

### NAT/Firewall Verification ✓
- [ ] Test outbound connectivity from clients
- [ ] Verify port forwarding works
- [ ] Check iptables rules are active
- [ ] Test firewall blocks unwanted traffic

## Phase 3: Testing & Documentation (60 min)

### Comprehensive Testing ✓
- [ ] **Network Connectivity**: Ping test between all machines
- [ ] **DNS Resolution**: Local and external queries work
- [ ] **Web Server Access**: HTTP redirects to HTTPS
- [ ] **TLS/SSL**: Certificate validation successful
- [ ] **NAT Functionality**: Internal machines can reach Internet
- [ ] **Firewall Rules**: Only necessary ports open
- [ ] **External Access**: Web server reachable from outside (if applicable)

### Security Verification ✓
- [ ] Run port scan from external: `nmap -sS [external-ip]`
- [ ] Verify SSL certificate: `openssl s_client -connect www.company.local:443`
- [ ] Check firewall rules: `iptables -L -n -v`
- [ ] Verify no unnecessary services running
- [ ] Check DNS security (no open recursion)

### Documentation ✓
- [ ] Network diagram drawn/printed
- [ ] IP address assignments documented
- [ ] Service configurations noted
- [ ] Security measures implemented listed
- [ ] Testing results recorded
- [ ] Any issues/workarounds documented

## Assignment Requirements Verification ✓

### Core Requirements ✓
- [ ] **Web server operational** and accessible over Internet via PROVIDED domain
- [ ] **TLS certificate** issued on SUBMISSION DATE (verify with `certbot certificates`)
- [ ] **Client workstation** can access both Internet and local web server
- [ ] **DNS server** operational and being used by all clients
- [ ] **NAT implementation** with single public IP

### Security Implementation ✓
- [ ] **Firewall rules** properly configured
- [ ] **HTTPS enforcement** (HTTP redirects)
- [ ] **DNS security** (restricted recursion)
- [ ] **Network segmentation** (internal vs external)
- [ ] **Minimal attack surface** (only necessary ports open)

## Emergency Backup Plans 🚨

### If Let's Encrypt Fails:
- [ ] Use self-signed certificate
- [ ] Document certificate warning handling

### If External Access Fails:
- [ ] Demonstrate local HTTPS access
- [ ] Show NAT configuration is correct
- [ ] Document network topology

### If DNS Fails:
- [ ] Use static `/etc/hosts` entries
- [ ] Demonstrate web server by IP address

### If Time Runs Short:
- [ ] Prioritize core functionality over advanced features
- [ ] Use grml5 as backup client if needed
- [ ] Focus on demonstration-ready components

## Quick Test Commands

### Network Tests
```bash
# Test all connectivity
ping 192.168.1.1 && ping 192.168.1.2 && ping 192.168.1.3 && ping 8.8.8.8

# Test DNS
nslookup www.company.local 192.168.1.2

# Test web server
curl -k https://www.company.local
```

### Security Tests
```bash
# CRITICAL: Security scan (mentioned in submission requirements)
nmap -sS [external-ip]  # Should show ONLY ports 80, 443, and SSH (if enabled)

# Check firewall rules
sudo iptables -L -n -v

# Check running services
ss -tuln

# Verify TLS certificate
openssl s_client -connect netseclab1.inf.tu-dresden.de:443 -servername netseclab1.inf.tu-dresden.de

# Verify certificate issue date
sudo certbot certificates | grep "Expiry Date"
```

## Time Management
- **Hour 1**: Physical setup + basic network config
- **Hour 2**: Complete network infrastructure
- **Hour 3**: DNS server setup and testing
- **Hour 4**: Web server basic setup
- **Hour 5**: TLS configuration and security
- **Hour 6**: Final testing and documentation

## Submission Requirements Check (Final 15 minutes)

### Team Preparation ✓
- [ ] **All team members** can explain network design and components
- [ ] **Architecture diagram** prepared and understood by all
- [ ] **Security measures** clearly documented and explainable
- [ ] **Troubleshooting** procedures known by team

### Final Technical Verification ✓
- [ ] **DNS server** actively used by all workstations (`nslookup` tests)
- [ ] **Web server** accessible from Internet via PROVIDED domain
- [ ] **Certificate date** matches submission date (`certbot certificates`)
- [ ] **Workstation** can reach external web servers AND internal server
- [ ] **Security scan** shows no unnecessary open ports (`nmap -sS [external-ip]`)

### Documentation & Demo Ready ✓
- [ ] All services running (`systemctl status`)
- [ ] Network connectivity verified
- [ ] Complete documentation ready
- [ ] Demo script rehearsed by team 