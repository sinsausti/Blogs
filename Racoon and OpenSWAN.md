# Raccoon and OpenS/WAN: The Complete IPsec VPN Guide

> **Legacy article:** Raccoon/ipsec-tools and OpenS/WAN are obsolete choices for a new deployment, and several historical examples below use algorithms such as 3DES, SHA-1, MD5, and old Diffie-Hellman groups that should not be deployed today. This article is retained for historical reference only. Use a maintained implementation such as strongSwan or Libreswan, prefer IKEv2, modern authenticated encryption, and current vendor guidance.

Hey network engineers and security enthusiasts! 🔐

In today's interconnected world, secure communication between networks is absolutely critical. Whether you're connecting branch offices, enabling remote access, or securing cloud communications, IPsec VPNs remain one of the most robust solutions available. Today we're diving deep into two powerful tools: **Raccoon** (the IKE daemon) and **OpenS/WAN** (the IPsec implementation).

Let's build some rock-solid VPN tunnels!

## Understanding the IPsec Ecosystem

Before we jump into configurations, let's understand what we're working with:

### What is IPsec?
IPsec (Internet Protocol Security) is a protocol suite that provides:
- **Authentication**: Verify the identity of communicating parties
- **Integrity**: Ensure data hasn't been tampered with
- **Confidentiality**: Encrypt data in transit
- **Anti-replay protection**: Prevent replay attacks

### The Key Players

**Raccoon**: 
- IKE (Internet Key Exchange) daemon from the KAME project
- Handles the negotiation of security associations (SAs)
- Manages authentication and key exchange
- Part of the ipsec-tools package

**OpenS/WAN**:
- Open Source implementation of IPsec
- Successor to FreeS/WAN
- Provides the kernel-level IPsec functionality
- Includes Pluto IKE daemon (alternative to Raccoon)

## Part 1: Installation and Initial Setup

### Installing Raccoon (ipsec-tools)

**On Ubuntu/Debian:**
```bash
# Install ipsec-tools (includes Raccoon)
sudo apt update
sudo apt install ipsec-tools

# Check installation
racoon -V
setkey -V
```

**On CentOS/RHEL:**
```bash
# Install ipsec-tools
sudo yum install ipsec-tools
# or on newer versions
sudo dnf install ipsec-tools
```

### Installing OpenS/WAN

**On Ubuntu/Debian:**
```bash
# Install OpenS/WAN
sudo apt install openswan

# Check installation
ipsec --version
```

**On CentOS/RHEL:**
```bash
# Install OpenS/WAN
sudo yum install openswan
# or
sudo dnf install libreswan  # Modern replacement
```

### Kernel IPsec Support

Most modern Linux kernels include IPsec support, but let's verify:

```bash
# Check for IPsec kernel modules
lsmod | grep -E "(esp|ah|xfrm)"

# Load modules if needed
sudo modprobe esp4
sudo modprobe ah4
sudo modprobe xfrm4_mode_tunnel
sudo modprobe xfrm_user
```

## Part 2: Raccoon Configuration Deep Dive

Raccoon's main configuration file is `/etc/racoon/racoon.conf`. Let's build a comprehensive setup.

### Basic Raccoon Configuration

```bash
# /etc/racoon/racoon.conf

# Global settings
path include "/etc/racoon";
path pre_shared_key "/etc/racoon/psk.txt";
path certificate "/etc/racoon/certs";

# Logging
log info;
log_file "/var/log/racoon.log";

# Listen on all interfaces
listen {
    isakmp 0.0.0.0[500];
    isakmp_natt 0.0.0.0[4500];  # NAT-T support
}

# Default timer settings
timer {
    counter 5;                # retransmission count
    interval 20 sec;          # interval between retransmissions
    persend 1;               # number of packets sent continuously
    phase1 30 sec;           # phase 1 timeout
    phase2 15 sec;           # phase 2 timeout
}

# Remote gateway configuration
remote 203.0.113.10 {
    exchange_mode main;       # or aggressive
    doi ipsec_doi;
    situation identity_only;
    
    # Phase 1 proposal
    proposal {
        encryption_algorithm 3des;     # or aes, blowfish
        hash_algorithm sha1;           # or md5
        authentication_method pre_shared_key;
        dh_group 2;                   # Diffie-Hellman group
    }
    
    # Generate policy automatically
    generate_policy on;
    
    # NAT traversal
    nat_traversal on;
    
    # Dead peer detection
    dpd_delay 30;
    dpd_retry 5;
    dpd_maxfail 3;
    
    # Phase 1 lifetime
    lifetime time 86400 sec;  # 24 hours
}

# Phase 2 (IPsec SA) configuration
sainfo address 192.168.1.0/24 any address 192.168.2.0/24 any {
    pfs_group 2;              # Perfect Forward Secrecy
    lifetime time 3600 sec;   # 1 hour
    encryption_algorithm 3des, aes;
    authentication_algorithm hmac_sha1, hmac_md5;
    compression_algorithm deflate;
}
```

### Pre-Shared Key Configuration

```bash
# /etc/racoon/psk.txt
# Format: remote_ip key

203.0.113.10    MyVerySecurePreSharedKey123!
10.0.0.1        AnotherSecureKey456@
```

**Secure the PSK file:**
```bash
sudo chmod 600 /etc/racoon/psk.txt
sudo chown root:root /etc/racoon/psk.txt
```

### Certificate-Based Authentication

For production environments, certificates are more secure:

```bash
# Generate CA certificate
openssl genrsa -out ca-key.pem 2048
openssl req -new -x509 -key ca-key.pem -out ca-cert.pem -days 3650

# Generate server certificate
openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem -out server-req.pem
openssl x509 -req -in server-req.pem -CA ca-cert.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -days 365

# Copy certificates to racoon directory
sudo cp ca-cert.pem /etc/racoon/certs/
sudo cp server-cert.pem /etc/racoon/certs/
sudo cp server-key.pem /etc/racoon/certs/
sudo chmod 600 /etc/racoon/certs/*
```

**Certificate-based racoon.conf:**
```bash
remote 203.0.113.10 {
    exchange_mode main;
    certificate_type x509 "server-cert.pem" "server-key.pem";
    ca_type x509 "ca-cert.pem";
    verify_cert on;
    send_cert on;
    send_cr on;
    
    proposal {
        encryption_algorithm aes;
        hash_algorithm sha1;
        authentication_method rsasig;  # RSA signature
        dh_group 2;
    }
}
```

### Advanced Raccoon Features

#### Multiple Remote Gateways

```bash
# Hub-and-spoke configuration
remote 203.0.113.10 {
    # Branch office A configuration
    exchange_mode main;
    proposal {
        encryption_algorithm aes;
        hash_algorithm sha256;
        authentication_method pre_shared_key;
        dh_group 14;
    }
}

remote 203.0.113.20 {
    # Branch office B configuration  
    exchange_mode main;
    proposal {
        encryption_algorithm aes;
        hash_algorithm sha256;
        authentication_method pre_shared_key;
        dh_group 14;
    }
}

# Corresponding SAinfo entries
sainfo address 192.168.1.0/24 any address 192.168.10.0/24 any {
    encryption_algorithm aes;
    authentication_algorithm hmac_sha256;
    lifetime time 3600 sec;
}

sainfo address 192.168.1.0/24 any address 192.168.20.0/24 any {
    encryption_algorithm aes;
    authentication_algorithm hmac_sha256;
    lifetime time 3600 sec;
}
```

#### XAuth (Extended Authentication)

For road warrior setups:

```bash
remote anonymous {
    exchange_mode aggressive;
    mode_cfg on;
    proposal {
        encryption_algorithm aes;
        hash_algorithm sha1;
        authentication_method xauth_psk_server;
        dh_group 2;
    }
}

mode_cfg {
    network4 192.168.100.0;
    netmask4 255.255.255.0;
    pool_size 50;
    dns4 8.8.8.8, 8.8.4.4;
    split_network include 192.168.0.0/16;
}
```

## Part 3: OpenS/WAN Configuration

OpenS/WAN uses `/etc/ipsec.conf` for configuration and includes the Pluto daemon as an alternative to Raccoon.

### Basic OpenS/WAN Setup

```bash
# /etc/ipsec.conf

# Global configuration
config setup
    # Debug logging (remove in production)
    plutodebug=all
    plutostderrlog=/var/log/pluto.log
    
    # Disable opportunistic encryption
    oe=off
    
    # Load connections at startup
    protostack=netkey

# Connection template
conn %default
    ikelifetime=24h
    keylife=1h
    rekeymargin=3m
    keyingtries=1
    keyexchange=ike
    compress=no

# Site-to-site connection
conn site-to-site
    type=tunnel
    authby=secret
    
    # Left side (local)
    left=192.168.1.1
    leftsubnet=192.168.1.0/24
    leftid=@site1.company.com
    
    # Right side (remote)
    right=203.0.113.10
    rightsubnet=192.168.2.0/24
    rightid=@site2.company.com
    
    # Algorithms
    ike=aes256-sha256-modp2048
    esp=aes256-sha256
    
    # Auto-start
    auto=start

# Road warrior connection
conn roadwarrior
    type=tunnel
    authby=secret
    
    # Server side
    left=203.0.113.1
    leftsubnet=192.168.0.0/16
    leftid=@vpn.company.com
    
    # Client side (dynamic)
    right=%any
    rightsourceip=192.168.100.0/24
    rightid=@%any
    
    # Auto-add for incoming connections
    auto=add
```

### OpenS/WAN Secrets File

```bash
# /etc/ipsec.secrets

# PSK for site-to-site
@site1.company.com @site2.company.com : PSK "MyVerySecurePreSharedKey123!"

# Road warrior PSKs
user1@company.com : PSK "User1SecurePassword"
user2@company.com : PSK "User2SecurePassword"

# RSA private key (for certificate auth)
: RSA /etc/ipsec.d/private/server.key "optional_passphrase"
```

### Certificate-Based OpenS/WAN

```bash
# Generate certificates (similar to Raccoon)
# Place files in appropriate directories:
# CA certificates: /etc/ipsec.d/cacerts/
# Server certificates: /etc/ipsec.d/certs/
# Private keys: /etc/ipsec.d/private/
# CRLs: /etc/ipsec.d/crls/

# Modified connection for certificates
conn site-to-site-cert
    type=tunnel
    authby=rsasig
    
    left=192.168.1.1
    leftcert=site1-cert.pem
    leftid="C=US, O=Company, CN=site1.company.com"
    
    right=203.0.113.10
    rightid="C=US, O=Company, CN=site2.company.com"
    
    auto=start
```

## Part 4: Policy Configuration with setkey

Both Raccoon and OpenS/WAN can work with manual policy configuration using `setkey`:

### Manual Policy Configuration

```bash
#!/bin/bash
# /etc/ipsec-policies.sh

# Flush existing policies and SAs
setkey -F
setkey -FP

# Define the networks
LOCAL_NET="192.168.1.0/24"
REMOTE_NET="192.168.2.0/24"
LOCAL_GW="192.168.1.1"
REMOTE_GW="203.0.113.10"

# Outbound policy (local to remote)
setkey -c << EOF
spdadd $LOCAL_NET $REMOTE_NET any -P out ipsec
    esp/tunnel/$LOCAL_GW-$REMOTE_GW/require;
EOF

# Inbound policy (remote to local)
setkey -c << EOF
spdadd $REMOTE_NET $LOCAL_NET any -P in ipsec
    esp/tunnel/$REMOTE_GW-$LOCAL_GW/require;
EOF

echo "IPsec policies configured"
```

### Dynamic Policy Management

```bash
# View current policies
setkey -DP

# View security associations
setkey -D

# Add specific SA manually (for testing)
setkey -c << EOF
add 192.168.1.1 203.0.113.10 esp 0x1001
    -E 3des-cbc "123456789012345678901234"
    -A hmac-sha1 "12345678901234567890";
EOF
```

## Part 5: Monitoring and Troubleshooting

### Essential Monitoring Commands

```bash
# Check Raccoon status
sudo racoon -F -f /etc/racoon/racoon.conf -l /var/log/racoon.log

# Check OpenS/WAN status
sudo ipsec status
sudo ipsec auto --status

# View active tunnels
ip xfrm state
ip xfrm policy

# Monitor real-time IPsec traffic
sudo tcpdump -i any esp
sudo tcpdump -i any port 500 or port 4500

# Check kernel IPsec statistics
cat /proc/net/xfrm_stat
```

### Log Analysis

**Raccoon logs:**
```bash
# Common log locations
tail -f /var/log/racoon.log
tail -f /var/log/syslog | grep racoon

# Important log patterns to watch for:
# "phase1 negotiation failed" - Authentication issues
# "phase2 negotiation failed" - Policy mismatches
# "DPD: remote seems to be dead" - Connectivity issues
```

**OpenS/WAN logs:**
```bash
# Pluto logs
tail -f /var/log/pluto.log
tail -f /var/log/auth.log | grep pluto

# Check connection status
ipsec auto --status | grep -A 5 "site-to-site"
```

### Common Issues and Solutions

#### Phase 1 Negotiation Failures

```bash
# Problem: Mismatched proposals
# Solution: Ensure both sides have compatible algorithms

# Raccoon debug mode
sudo racoon -F -d -f /etc/racoon/racoon.conf

# Check proposals on both sides match:
# - Encryption algorithm (3des, aes)
# - Hash algorithm (md5, sha1, sha256)
# - DH group (1, 2, 5, 14, etc.)
# - Authentication method (psk, rsasig)
```

#### Phase 2 Negotiation Failures

```bash
# Problem: Policy mismatches
# Check that subnet definitions match on both sides

# Debug with setkey
setkey -DP | grep -A 3 -B 3 "192.168"

# Verify network reachability
ping -c 3 203.0.113.10

# Check firewall rules
iptables -L -n | grep -E "(500|4500|esp|ah)"
```

#### NAT Traversal Issues

```bash
# Enable NAT-T in Raccoon
nat_traversal on;

# OpenS/WAN NAT-T
nat_traversal=yes

# Check if NAT-T is working
netstat -un | grep 4500

# Force NAT-T for testing
# In OpenS/WAN: forceencaps=yes
```

### Performance Monitoring

```bash
# Monitor IPsec throughput
iftop -i ipsec0  # if using ipsec interface

# Check CPU usage during encryption
top -p $(pgrep racoon)
iostat 1

# Network interface statistics
ip -s link show
cat /proc/net/dev
```

## Part 6: Advanced Configurations

### High Availability Setup

#### Raccoon with Keepalived

```bash
# /etc/keepalived/keepalived.conf
vrrp_script chk_racoon {
    script "/bin/pgrep racoon"
    interval 2
    weight 2
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mypassword
    }
    virtual_ipaddress {
        192.168.1.100
    }
    track_script {
        chk_racoon
    }
}
```

#### OpenS/WAN Failover

```bash
# Primary connection
conn primary
    left=192.168.1.1
    right=203.0.113.10
    auto=start

# Backup connection
conn backup
    left=192.168.1.1
    right=203.0.113.20
    auto=start
    priority=10  # Lower priority
```

### Scripted Automation

#### Connection Health Check Script

```bash
#!/bin/bash
# /usr/local/bin/vpn-healthcheck.sh

VPN_REMOTE="203.0.113.10"
VPN_TEST_HOST="192.168.2.100"
LOG_FILE="/var/log/vpn-health.log"

check_tunnel() {
    # Check if tunnel is up
    if ip xfrm state | grep -q "$VPN_REMOTE"; then
        echo "$(date): Tunnel to $VPN_REMOTE is UP" >> $LOG_FILE
        
        # Test connectivity through tunnel
        if ping -c 3 -W 5 $VPN_TEST_HOST &>/dev/null; then
            echo "$(date): Connectivity test PASSED" >> $LOG_FILE
            return 0
        else
            echo "$(date): Connectivity test FAILED" >> $LOG_FILE
            return 1
        fi
    else
        echo "$(date): Tunnel to $VPN_REMOTE is DOWN" >> $LOG_FILE
        return 1
    fi
}

restart_tunnel() {
    echo "$(date): Restarting VPN tunnel" >> $LOG_FILE
    
    # For Raccoon
    systemctl restart racoon
    
    # For OpenS/WAN
    # ipsec auto --down site-to-site
    # ipsec auto --up site-to-site
}

# Main health check
if ! check_tunnel; then
    restart_tunnel
    sleep 30
    if check_tunnel; then
        echo "$(date): Tunnel recovery SUCCESSFUL" >> $LOG_FILE
    else
        echo "$(date): Tunnel recovery FAILED - manual intervention required" >> $LOG_FILE
        # Send alert email/notification here
    fi
fi
```

#### Automatic Certificate Renewal

```bash
#!/bin/bash
# /usr/local/bin/cert-renewal.sh

CERT_DIR="/etc/racoon/certs"
CERT_FILE="$CERT_DIR/server-cert.pem"
DAYS_BEFORE_EXPIRY=30

# Check certificate expiration
if openssl x509 -checkend $((DAYS_BEFORE_EXPIRY * 24 * 3600)) -noout -in $CERT_FILE; then
    echo "Certificate is still valid"
    exit 0
fi

echo "Certificate expires soon, renewing..."

# Generate new certificate request
openssl req -new -key $CERT_DIR/server-key.pem -out $CERT_DIR/server-new-req.pem \
    -subj "/C=US/O=Company/CN=vpn.company.com"

# Sign with CA (assuming automated CA)
openssl x509 -req -in $CERT_DIR/server-new-req.pem \
    -CA $CERT_DIR/ca-cert.pem -CAkey $CERT_DIR/ca-key.pem \
    -CAcreateserial -out $CERT_DIR/server-new-cert.pem -days 365

# Backup old certificate
mv $CERT_FILE $CERT_DIR/server-cert-old.pem

# Install new certificate
mv $CERT_DIR/server-new-cert.pem $CERT_FILE

# Restart services
systemctl restart racoon
# or for OpenS/WAN: systemctl restart ipsec

echo "Certificate renewed successfully"
```

## Part 7: Security Best Practices

### Hardening Your IPsec Configuration

#### Strong Cryptographic Algorithms

```bash
# Raccoon - Use strong algorithms
proposal {
    encryption_algorithm aes256;    # Use AES-256
    hash_algorithm sha256;          # Use SHA-256 or better
    authentication_method pre_shared_key;
    dh_group 14;                   # Use DH group 14 or higher
}

# OpenS/WAN - Modern cipher suites
ike=aes256-sha256-modp2048,aes256-sha256-modp3072
esp=aes256-sha256,aes256-sha512
```

#### Perfect Forward Secrecy

```bash
# Enable PFS in Raccoon
sainfo address 192.168.1.0/24 any address 192.168.2.0/24 any {
    pfs_group 14;               # Use DH group 14 for PFS
    encryption_algorithm aes256;
    authentication_algorithm hmac_sha256;
}

# PFS in OpenS/WAN (enabled by default with proper DH groups)
pfs=yes
```

#### Regular Key Rotation

```bash
# Short SA lifetimes for better security
# Raccoon
lifetime time 3600 sec;     # 1 hour for Phase 2
lifetime time 86400 sec;    # 24 hours for Phase 1

# OpenS/WAN
keylife=1h
ikelifetime=24h
```

### Firewall Integration

```bash
#!/bin/bash
# IPsec-aware firewall rules

# Allow IKE (UDP 500)
iptables -A INPUT -p udp --dport 500 -j ACCEPT
iptables -A OUTPUT -p udp --sport 500 -j ACCEPT

# Allow NAT-T (UDP 4500)
iptables -A INPUT -p udp --dport 4500 -j ACCEPT
iptables -A OUTPUT -p udp --sport 4500 -j ACCEPT

# Allow ESP protocol
iptables -A INPUT -p esp -j ACCEPT
iptables -A OUTPUT -p esp -j ACCEPT

# Allow AH protocol (if used)
iptables -A INPUT -p ah -j ACCEPT
iptables -A OUTPUT -p ah -j ACCEPT

# Allow decrypted traffic from VPN subnets
iptables -A INPUT -s 192.168.2.0/24 -j ACCEPT
iptables -A OUTPUT -d 192.168.2.0/24 -j ACCEPT

# Prevent IPsec traffic from bypassing encryption
iptables -A OUTPUT -d 192.168.2.0/24 -m policy --dir out --pol none -j DROP
```

## Part 8: Migration and Interoperability

### Migrating from Raccoon to OpenS/WAN

```bash
# Convert Raccoon configuration to OpenS/WAN format

# Raccoon remote section:
# remote 203.0.113.10 {
#     proposal {
#         encryption_algorithm aes;
#         hash_algorithm sha1;
#         dh_group 2;
#     }
# }

# Equivalent OpenS/WAN:
conn migrated-connection
    left=%defaultroute
    right=203.0.113.10
    ike=aes-sha1-modp1024
    esp=aes-sha1
    authby=secret
    auto=start
```

### Interoperability with Other Vendors

#### Common Vendor-Specific Settings

```bash
# Cisco ASA compatibility
conn cisco-asa
    # Cisco often uses aggressive mode
    aggrmode=yes
    # Specific lifetime values
    ikelifetime=86400s
    keylife=28800s
    # DPD settings for Cisco
    dpddelay=10
    dpdtimeout=60

# Juniper compatibility  
conn juniper-srx
    # Juniper-specific DH groups
    ike=aes256-sha1-modp1536
    # Specific rekey margins
    rekeymargin=540s

# pfSense compatibility
conn pfsense
    # Often needs specific fragmentation settings
    fragmentation=yes
    # May require specific NAT-T settings
    nat_traversal=yes
```

## Part 9: Performance Optimization

### Tuning for High Throughput

#### Kernel Parameters

```bash
# /etc/sysctl.conf optimizations
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 65536 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# IPsec-specific tuning
net.ipv4.ip_no_pmtu_disc = 1
net.ipv4.tcp_mtu_probing = 1

# Apply changes
sysctl -p
```

#### Hardware Acceleration

```bash
# Check for crypto acceleration
cat /proc/crypto | grep -E "(aes|sha)"

# Intel AES-NI support
grep -i aes /proc/cpuinfo

# Load hardware crypto modules if available
modprobe aesni_intel
modprobe ghash_clmulni_intel
```

#### Multi-Core Optimization

```bash
# Raccoon with multiple worker processes
# (Limited support - consider using OpenS/WAN for better SMP)

# OpenS/WAN can better utilize multiple cores
# through kernel netkey stack optimization
echo 'net.core.netdev_max_backlog = 5000' >> /etc/sysctl.conf
```

## Wrapping Up

Raccoon and OpenS/WAN represent proven, battle-tested solutions for IPsec VPN deployments. While newer solutions like strongSwan and WireGuard are gaining popularity, these tools remain relevant for many enterprise environments.

### Key Takeaways

**When to Use Raccoon:**
- Need fine-grained control over IKE negotiations
- Working with legacy systems that require specific IKE behaviors
- Comfortable with manual policy management

**When to Use OpenS/WAN:**
- Want integrated solution with less complexity  
- Need better vendor interoperability
- Require robust certificate management

**Best Practices to Remember:**
- Always use strong cryptographic algorithms
- Implement proper monitoring and alerting
- Keep certificates and PSKs secure
- Regular security audits and updates
- Test failover scenarios regularly
- Document your configurations thoroughly

### The Future of IPsec

While IPsec remains crucial for enterprise networking, consider these trends:
- **WireGuard**: Simpler, more performant for many use cases
- **strongSwan**: More modern IPsec implementation
- **Cloud-native VPNs**: Managed solutions reducing operational overhead

However, Raccoon and OpenS/WAN still excel in environments requiring:
- Specific compliance requirements
- Integration with legacy systems
- Fine-tuned performance optimization
- Custom authentication mechanisms

The investment in understanding these tools pays dividends in network security expertise and the ability to troubleshoot complex VPN issues that inevitably arise in production environments.

Have you deployed Raccoon or OpenS/WAN in production? What challenges did you face, and how did you overcome them? Share your experiences in the comments below!

---

*May your tunnels be secure and your packets flow freely! 🔒🚀*
