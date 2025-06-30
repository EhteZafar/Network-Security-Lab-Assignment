# Network Security Lab - Detailed Setup Guide

## Machine Roles Overview
- **grml1**: Router/NAT Gateway (192.168.1.1)
- **grml2**: DNS Server (192.168.1.2)
- **grml3**: Web Server (192.168.1.3)
- **grml4**: Client Workstation (192.168.1.4)
- **grml5**: Client Workstation (192.168.1.5)
- **grml6**: Management/Backup (192.168.1.6)

---

## grml1: Router/NAT Gateway Configuration

### 1. Network Interface Configuration
```bash
# Configure external interface (assuming eth0 for external, eth1 for internal)
sudo ip addr add 192.168.1.1/24 dev eth1
sudo ip link set eth1 up

# Enable IP forwarding
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 2. NAT and Firewall Rules
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

# NAT for internal network
sudo iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

# Allow internal to external
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# Port forwarding for web server (HTTP and HTTPS)
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.3:80
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 443 -j DNAT --to-destination 192.168.1.3:443

# Allow forwarded connections to web server
sudo iptables -A FORWARD -d 192.168.1.3 -p tcp --dport 80 -j ACCEPT
sudo iptables -A FORWARD -d 192.168.1.3 -p tcp --dport 443 -j ACCEPT

# Allow SSH for management
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow DNS from internal network
sudo iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 53 -j ACCEPT
sudo iptables -A INPUT -s 192.168.1.0/24 -p udp --dport 53 -j ACCEPT

# Default policies
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

### 2. Configure Network Interface
```bash
sudo ip addr add 192.168.1.2/24 dev eth0
sudo ip route add default via 192.168.1.1
```

### 3. Configure BIND9
```bash
# Edit /etc/bind/named.conf.options
sudo tee /etc/bind/named.conf.options > /dev/null <<EOF
options {
    directory "/var/cache/bind";
    
    // Listen on internal interface
    listen-on { 127.0.0.1; 192.168.1.2; };
    
    // Allow queries from internal network
    allow-query { localhost; 192.168.1.0/24; };
    
    // Disable recursion for external queries (security)
    allow-recursion { localhost; 192.168.1.0/24; };
    
    // Forwarders for external DNS
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
    
    // Enable DNSSEC
    dnssec-validation auto;
    
    auth-nxdomain no;
};
EOF
```

### 4. Create Local Zone
```bash
# Edit /etc/bind/named.conf.local
sudo tee /etc/bind/named.conf.local > /dev/null <<EOF
zone "company.local" {
    type master;
    file "/etc/bind/db.company.local";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};
EOF
```

### 5. Create Zone Files
```bash
# Forward zone file
sudo tee /etc/bind/db.company.local > /dev/null <<EOF
\$TTL    604800
@       IN      SOA     ns1.company.local. admin.company.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.company.local.
@       IN      A       192.168.1.3
ns1     IN      A       192.168.1.2
www     IN      A       192.168.1.3
web     IN      A       192.168.1.3
gateway IN      A       192.168.1.1
client1 IN      A       192.168.1.4
client2 IN      A       192.168.1.5
EOF

# Reverse zone file
sudo tee /etc/bind/db.192.168.1 > /dev/null <<EOF
\$TTL    604800
@       IN      SOA     ns1.company.local. admin.company.local. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.company.local.
1       IN      PTR     gateway.company.local.
2       IN      PTR     ns1.company.local.
3       IN      PTR     www.company.local.
4       IN      PTR     client1.company.local.
5       IN      PTR     client2.company.local.
EOF
```

### 6. Start and Enable BIND9
```bash
sudo systemctl restart bind9
sudo systemctl enable bind9
```

---

## grml3: Web Server Configuration

### 1. Configure Network Interface
```bash
sudo ip addr add 192.168.1.3/24 dev eth0
sudo ip route add default via 192.168.1.1
echo "nameserver 192.168.1.2" | sudo tee /etc/resolv.conf
```

### 2. Install Apache2 and Dependencies
```bash
sudo apt update
sudo apt install apache2 certbot python3-certbot-apache -y
```

### 3. Create Basic Website
```bash
sudo tee /var/www/html/index.html > /dev/null <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>Company Web Server</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .header { background-color: #f0f0f0; padding: 20px; border-radius: 5px; }
        .content { margin-top: 20px; }
        .security { background-color: #e8f5e8; padding: 15px; border-radius: 5px; margin-top: 20px; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Welcome to Company Web Server</h1>
        <p>Network Security Lab Assignment</p>
    </div>
    
    <div class="content">
        <h2>Server Information</h2>
        <ul>
            <li>Server IP: 192.168.1.3</li>
            <li>Domain: www.company.local</li>
            <li>TLS/SSL: Enabled</li>
            <li>DNS Server: 192.168.1.2</li>
        </ul>
    </div>
    
    <div class="security">
        <h3>Security Features</h3>
        <ul>
            <li>HTTPS/TLS Encryption</li>
            <li>NAT Protection</li>
            <li>Firewall Rules</li>
            <li>Local DNS Resolution</li>
        </ul>
    </div>
</body>
</html>
EOF
```

### 4. Configure Apache Virtual Host
```bash
sudo tee /etc/apache2/sites-available/company.conf > /dev/null <<EOF
<VirtualHost *:80>
    ServerName www.company.local
    ServerAlias company.local
    DocumentRoot /var/www/html
    
    # Redirect to HTTPS
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
</VirtualHost>

<VirtualHost *:443>
    ServerName www.company.local
    ServerAlias company.local
    DocumentRoot /var/www/html
    
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
    SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key
    
    # Security headers
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set X-Content-Type-Options nosniff
    Header always set X-Frame-Options DENY
    Header always set X-XSS-Protection "1; mode=block"
</VirtualHost>
EOF
```

### 5. Enable Modules and Site
```bash
sudo a2enmod ssl
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2ensite company.conf
sudo a2dissite 000-default.conf
sudo systemctl restart apache2
```

### 6. Set up Let's Encrypt (if external domain available)
```bash
# For external domain (replace with actual domain)
# sudo certbot --apache -d yourdomain.com

# For local testing with self-signed certificate
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/company.local.key \
    -out /etc/ssl/certs/company.local.crt \
    -subj "/C=DE/ST=Saxony/L=Dresden/O=Company/CN=www.company.local"

# Update virtual host with new certificate
sudo sed -i 's|ssl-cert-snakeoil.pem|company.local.crt|g' /etc/apache2/sites-available/company.conf
sudo sed -i 's|ssl-cert-snakeoil.key|company.local.key|g' /etc/apache2/sites-available/company.conf
sudo systemctl restart apache2
```

---

## grml4 & grml5: Client Workstation Configuration

### 1. Configure Network Interface (grml4)
```bash
sudo ip addr add 192.168.1.4/24 dev eth0
sudo ip route add default via 192.168.1.1
echo "nameserver 192.168.1.2" | sudo tee /etc/resolv.conf
```

### 2. Configure Network Interface (grml5)
```bash
sudo ip addr add 192.168.1.5/24 dev eth0
sudo ip route add default via 192.168.1.1
echo "nameserver 192.168.1.2" | sudo tee /etc/resolv.conf
```

### 3. Install Web Browser and Tools
```bash
sudo apt update
sudo apt install firefox-esr curl wget nmap dig -y
```

### 4. Test Commands
```bash
# Test DNS resolution
nslookup www.company.local
nslookup google.com

# Test web server access
curl -k https://www.company.local
curl -k https://192.168.1.3

# Test external connectivity
ping 8.8.8.8
curl https://www.google.com
```

---

## grml6: Management/Backup Configuration

### 1. Configure Network Interface
```bash
sudo ip addr add 192.168.1.6/24 dev eth0
sudo ip route add default via 192.168.1.1
echo "nameserver 192.168.1.2" | sudo tee /etc/resolv.conf
```

### 2. Install Monitoring Tools
```bash
sudo apt update
sudo apt install nmap tcpdump wireshark-common ngrep netstat-nat -y
```

### 3. Network Monitoring Scripts
```bash
# Create monitoring script
cat > ~/network_monitor.sh << 'EOF'
#!/bin/bash
echo "=== Network Status Monitoring ==="
echo "Date: $(date)"
echo
echo "=== DNS Resolution Test ==="
nslookup www.company.local 192.168.1.2
echo
echo "=== Web Server Connectivity ==="
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://192.168.1.3
echo
echo "=== External Connectivity ==="
ping -c 3 8.8.8.8
echo
echo "=== Active Connections ==="
netstat -tuln
EOF

chmod +x ~/network_monitor.sh
```

---

## Testing and Validation

### 1. Network Connectivity Tests
```bash
# From any machine, test ping to all others
ping 192.168.1.1  # Gateway
ping 192.168.1.2  # DNS
ping 192.168.1.3  # Web server
ping 192.168.1.4  # Client 1
ping 192.168.1.5  # Client 2

# Test external connectivity
ping 8.8.8.8
```

### 2. DNS Tests
```bash
# From client machines
nslookup www.company.local
nslookup google.com
dig @192.168.1.2 www.company.local
```

### 3. Web Server Tests
```bash
# HTTP access (should redirect to HTTPS)
curl -v http://www.company.local

# HTTPS access
curl -k https://www.company.local

# From external (if accessible)
curl -k https://[external-ip]
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