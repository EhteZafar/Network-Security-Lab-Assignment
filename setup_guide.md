# Network Security Lab - Detailed Setup Guide

## Machine Roles Overview (DMZ Architecture)
- **grml1**: NAT Gateway Machine (Public: 141.76.46.220, DMZ: 192.168.2.1, Internal: 192.168.1.1)
- **grml2**: DNS Server (192.168.2.2) - **DMZ Network**
- **grml3**: Web Server (192.168.2.10) - **DMZ Network**
- **grml4**: Client Workstation (192.168.1.4) - Internal Network
- **grml5**: Client Workstation (192.168.1.5) - Internal Network

**Network Segmentation:**
- **External**: 141.76.46.0/24 (Public Internet)
- **DMZ**: 192.168.2.0/24 (Web Server, DNS Server - Isolated)
- **Internal**: 192.168.1.0/24 (Client Workstations)

---

## grml1: NAT Gateway Configuration

### 1. Network Interface Configuration (Three-Zone Setup)
```bash
# Configure external interface (use any available port)
sudo ip addr add 141.76.46.220/24 dev eth0
sudo ip route add default via 141.76.46.254
sudo ip link set eth0 up

# Configure DMZ interface (connects to DMZ switch)
sudo ip addr add 192.168.2.1/24 dev eth1
sudo ip link set eth1 up

# Configure internal interface (connects to Internal switch)
sudo ip addr add 192.168.1.1/24 dev eth2
sudo ip link set eth2 up

# Enable IP forwarding
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 2. DMZ Firewall Rules (Three-Zone Security)
```bash
# Clear existing rules
sudo iptables -F
sudo iptables -t nat -F

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# NAT for both networks
sudo iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE  # Internal
sudo iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -o eth0 -j MASQUERADE  # DMZ

# PORT FORWARDING: External to DMZ Web Server
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.2.10:80
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j DNAT --to-destination 192.168.2.10:443

# EXTERNAL to DMZ: Allow web traffic only
sudo iptables -A FORWARD -i eth0 -o eth1 -d 192.168.2.10 -p tcp --dport 80 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -d 192.168.2.10 -p tcp --dport 443 -j ACCEPT

# INTERNAL to EXTERNAL: Allow all outbound
sudo iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT

# INTERNAL to DMZ: Allow web server access only
sudo iptables -A FORWARD -i eth2 -o eth1 -d 192.168.2.10 -p tcp --dport 80 -j ACCEPT
sudo iptables -A FORWARD -i eth2 -o eth1 -d 192.168.2.10 -p tcp --dport 443 -j ACCEPT

# DMZ to EXTERNAL: Allow web server outbound (for Let's Encrypt, updates)
sudo iptables -A FORWARD -i eth1 -o eth0 -s 192.168.2.10 -j ACCEPT

# DMZ to EXTERNAL: Allow DNS server outbound (for DNS forwarding to 141.30.1.1, updates)
sudo iptables -A FORWARD -i eth1 -o eth0 -s 192.168.2.2 -p udp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -s 192.168.2.2 -p tcp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -s 192.168.2.2 -p tcp --dport 80 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -s 192.168.2.2 -p tcp --dport 443 -j ACCEPT

# INTERNAL to DMZ: Allow DNS queries to DNS server
sudo iptables -A FORWARD -i eth2 -o eth1 -d 192.168.2.2 -p tcp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -i eth2 -o eth1 -d 192.168.2.2 -p udp --dport 53 -j ACCEPT

# Management access (SSH from internal only)
sudo iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 22 -j ACCEPT

# Default policies (deny all other traffic)
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Save rules
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### 3. Enable IP Forwarding Permanently
```bash
# Edit /etc/sysctl.conf
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
```

---

## grml2: DNS Server Configuration

### 1. Install BIND9
```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc -y
```

### 2. Configure Network Interface (DMZ)
```bash
sudo ip addr add 192.168.2.2/24 dev eth1
sudo ip route add default via 192.168.2.1
sudo ip link set eth1 up
```

### 3. Configure BIND9
```bash
# Edit /etc/bind/named.conf.options
sudo tee /etc/bind/named.conf.options > /dev/null <<EOF
options {
    directory "/var/cache/bind";
    
    // Listen on DMZ interface
    listen-on { 127.0.0.1; 192.168.2.2; };
    
    // Allow queries from internal network and DMZ
    allow-query { localhost; 192.168.1.0/24; 192.168.2.0/24; };
    
    // Allow recursion for internal and DMZ networks
    allow-recursion { localhost; 192.168.1.0/24; 192.168.2.0/24; };
    
    // Forwarders for external DNS (use ZIH DNS as recommended)
    forwarders {
        141.30.1.1;    // ZIH DNS server
        8.8.8.8;
        8.8.4.4;
    };
    
    // Enable DNSSEC
    dnssec-validation auto;
    
    auth-nxdomain no;
};
EOF
```

### 4. Create Local Zones (Simplified for your setup)
```bash
# Edit /etc/bind/named.conf.local
sudo tee /etc/bind/named.conf.local > /dev/null <<EOF
zone "lab.local" {
    type master;
    file "/etc/bind/db.lab.local";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};

zone "2.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.2";
};
EOF
```

### 5. Create Zone Files (Simplified)
```bash
# Forward zone file - simple internal names
sudo tee /etc/bind/db.lab.local > /dev/null <<EOF
\$TTL    604800
@       IN      SOA     dns.lab.local. admin.lab.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.lab.local.
dns     IN      A       192.168.2.2
web     IN      A       192.168.2.10
gateway IN      A       192.168.1.1
client1 IN      A       192.168.1.4
client2 IN      A       192.168.1.5
; Add alias for your actual domain (internal resolution)
netseclab   IN  A       192.168.2.10
EOF

# Reverse zone file for Internal network (192.168.1.x)
sudo tee /etc/bind/db.192.168.1 > /dev/null <<EOF
\$TTL    604800
@       IN      SOA     dns.lab.local. admin.lab.local. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.lab.local.
1       IN      PTR     gateway.lab.local.
4       IN      PTR     client1.lab.local.
5       IN      PTR     client2.lab.local.
EOF

# Reverse zone file for DMZ network (192.168.2.x)
sudo tee /etc/bind/db.192.168.2 > /dev/null <<EOF
\$TTL    604800
@       IN      SOA     dns.lab.local. admin.lab.local. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.lab.local.
2       IN      PTR     dns.lab.local.
10      IN      PTR     web.lab.local.
EOF
```

### 6. Start and Enable BIND9
```bash
sudo systemctl restart bind9
sudo systemctl enable bind9
```

---

## grml3: Web Server Configuration

### 1. Configure Network Interface (DMZ)
```bash
sudo ip addr add 192.168.2.10/24 dev eth1
sudo ip route add default via 192.168.2.1
sudo ip link set eth1 up
echo "nameserver 192.168.2.2" | sudo tee /etc/resolv.conf
echo "nameserver 141.30.1.1" | sudo tee -a /etc/resolv.conf
```

### 2. Install Apache2 and Dependencies (Your actual setup)
```bash
sudo apt update
sudo apt install apache2 certbot python3-certbot-apache -y
```

### 3. Create Basic Website (Optional - you can use default Apache page)
```bash
# Create a simple website or use the default Apache page
sudo tee /var/www/html/index.html > /dev/null <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>Network Security Lab Web Server</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .header { background-color: #f0f0f0; padding: 20px; border-radius: 5px; }
        .content { margin-top: 20px; }
        .security { background-color: #e8f5e8; padding: 15px; border-radius: 5px; margin-top: 20px; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Network Security Lab Web Server</h1>
        <p>TU Dresden - Network Security Assignment</p>
    </div>
    
    <div class="content">
        <h2>Server Information</h2>
        <ul>
            <li>Server IP: 192.168.2.10 (DMZ)</li>
            <li>Domain: netseclab1.inf.tu-dresden.de</li>
            <li>TLS/SSL: Enabled (Let's Encrypt)</li>
            <li>DNS Server: 192.168.2.2 + 141.30.1.1</li>
        </ul>
    </div>
    
    <div class="security">
        <h3>Security Features</h3>
        <ul>
            <li>HTTPS/TLS Encryption</li>
            <li>NAT Protection</li>
            <li>Firewall Rules</li>
            <li>DMZ Network Segmentation</li>
        </ul>
    </div>
</body>
</html>
EOF
```

### 4. Set up Let's Encrypt SSL Certificate (Your actual setup)
```bash
# This is what you actually ran:
sudo certbot --apache -d netseclab1.inf.tu-dresden.de

# Verify certificate installation
sudo certbot certificates

# Check Apache configuration
sudo apache2ctl configtest

# Restart Apache
sudo systemctl restart apache2
sudo systemctl enable apache2
```

### 5. Verify Your Setup
```bash
# Test HTTPS locally
curl -k https://localhost

# Check certificate
openssl s_client -connect localhost:443 -servername netseclab1.inf.tu-dresden.de

# Check Apache status
sudo systemctl status apache2

# View current sites
sudo apache2ctl -S
```

---

## grml4 & grml5: Client Workstation Configuration

### 1. Configure Network Interface (grml4)
```bash
sudo ip addr add 192.168.1.4/24 dev eth1
sudo ip route add default via 192.168.1.1
sudo ip link set eth1 up
echo "nameserver 192.168.2.2" | sudo tee /etc/resolv.conf
```

### 2. Configure Network Interface (grml5)
```bash
sudo ip addr add 192.168.1.5/24 dev eth1
sudo ip route add default via 192.168.1.1
sudo ip link set eth1 up
echo "nameserver 192.168.2.2" | sudo tee /etc/resolv.conf
```

### 3. Install Web Browser and Tools
```bash
sudo apt update
sudo apt install firefox-esr curl wget nmap dig -y
```

### 4. Test Commands (Updated for your setup)
```bash
# Test DNS resolution for internal names
nslookup web.lab.local
nslookup google.com

# Test your web server
curl -k https://netseclab1.inf.tu-dresden.de
curl -k https://192.168.2.10

# Test external connectivity
ping 8.8.8.8
curl https://www.google.com
```

---

## Testing and Validation

### 1. Network Connectivity Tests
```bash
# From any machine, test ping to all others
ping 192.168.1.1  # Gateway
ping 192.168.2.2  # DNS (DMZ)
ping 192.168.2.10 # Web server (DMZ)
ping 192.168.1.4  # Client 1
ping 192.168.1.5  # Client 2

# Test external connectivity
ping 8.8.8.8
```

### 2. DNS Tests (Updated)
```bash
# From client machines
nslookup web.lab.local
nslookup google.com
dig @192.168.2.2 web.lab.local
```

### 3. Web Server Tests (Updated for your actual domain)
```bash
# Test your actual domain (should work with Let's Encrypt certificate)
curl -I https://netseclab1.inf.tu-dresden.de

# Test by IP address
curl -k https://192.168.2.10

# Test from external internet (if accessible)
curl -I https://netseclab1.inf.tu-dresden.de
```

### 4. Security Verification
```bash
# Check firewall rules
sudo iptables -L -n -v

# Check SSL certificate
openssl s_client -connect www.company.local:443 -servername www.company.local

# Port scan from external
nmap -sS [external-ip]
```

## Troubleshooting Commands

```bash
# Check service status
sudo systemctl status bind9
sudo systemctl status apache2

# Check logs
sudo journalctl -u bind9 -f
sudo tail -f /var/log/apache2/error.log

# Network troubleshooting
ip route show
ip addr show
iptables -L -n -v
``` 