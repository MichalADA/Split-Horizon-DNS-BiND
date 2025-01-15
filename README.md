# Split-Horizon DNS Configuration on CentOS/AlmaLinux

This guide walks you through configuring Split-Horizon DNS using BIND on CentOS or AlmaLinux. Split-Horizon DNS allows a DNS server to return different records depending on the source of the query (internal vs. external network).

## **1. Prerequisites**
- CentOS or AlmaLinux or Redhatt with root or sudo access.
- Installed BIND (`bind` and `bind-utils`).

## **2. Install BIND**
```bash
sudo dnf install bind bind-utils -y
```

## **3. Prepare Zone Files**

### Create the Zone Directory
```bash
sudo mkdir -p /etc/named/zones
```

### Internal Zone File
Create the file `/etc/named/zones/internal.example.com.zone`:
```bash
$TTL 86400
@   IN  SOA   ns1.example.com. admin.example.com. (
            2025011501 ; Serial
            3600       ; Refresh
            1800       ; Retry
            1209600    ; Expire
            86400 )    ; Minimum TTL
@   IN  NS    ns1.example.com.
ns1 IN  A     192.168.1.1
www IN  A     192.168.1.2
mail IN  A     192.168.1.3
```

### External Zone File
Create the file `/etc/named/zones/external.example.com.zone`:
```bash
$TTL 86400
@   IN  SOA   ns1.example.com. admin.example.com. (
            2025011501 ; Serial
            3600       ; Refresh
            1800       ; Retry
            1209600    ; Expire
            86400 )    ; Minimum TTL
@   IN  NS    ns1.example.com.
ns1 IN  A     203.0.113.1
www IN  A     203.0.113.2
mail IN  A     203.0.113.3
```

## **4. Configure BIND**
Edit the `/etc/named.conf` file to define views for internal and external queries.

### Internal View
```plaintext
view "internal" {
    match-clients { 192.168.1.0/24; localhost; }; // Internal network
    zone "example.com" {
        type master;
        file "/etc/named/zones/internal.example.com.zone";
    };
};
```

### External View
```plaintext
view "external" {
    match-clients { any; }; // All other sources
    zone "example.com" {
        type master;
        file "/etc/named/zones/external.example.com.zone";
    };
};
```

## **5. Validate Configuration**

### Check the Configuration File
```bash
sudo named-checkconf
```

### Check Zone Files
```bash
sudo named-checkzone example.com /etc/named/zones/internal.example.com.zone
sudo named-checkzone example.com /etc/named/zones/external.example.com.zone
```

## **6. Restart BIND**
Enable and restart the BIND service:
```bash
sudo systemctl enable named
sudo systemctl restart named
```
Check the status:
```bash
sudo systemctl status named
```

## **7. Configure Firewall**
Allow DNS traffic through the firewall:
```bash
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload
```

## **8. Testing**

### Internal Network Test
Run this command from a machine in the internal network:
```bash
dig @localhost www.example.com
```
Expected result: IP from `internal.example.com.zone` (e.g., `192.168.1.2`).

### External Network Test
Run this command from an external machine:
```bash
dig @<PUBLIC_IP_OF_BIND_SERVER> www.example.com
```
Expected result: IP from `external.example.com.zone` (e.g., `203.0.113.2`).

## **9. Optional: Enable Logging**

### Add Logging Configuration
Edit `/etc/named.conf` to add logging:
```plaintext
logging {
    channel query_log {
        file "/var/log/named_queries.log";
        severity dynamic;
    };
    category queries { query_log; };
};
```

### Create Log File
```bash
sudo touch /var/log/named_queries.log
sudo chmod 640 /var/log/named_queries.log
sudo chown named:named /var/log/named_queries.log
```

### Restart BIND
```bash
sudo systemctl restart named
```

---

 plit-Horizon DNS server is now configured and ready to use. If you encounter any issues, double-check your configuration files and logs for troubleshooting.
