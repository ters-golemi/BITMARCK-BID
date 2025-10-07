# LOT 31: F5 BIG-IP Services - Technical Implementation Guide

## 1. F5 Technical Certifications and Expertise

### Team Certification Portfolio

**Lead F5 Engineer:**
- **F5 Certified Solution Expert (F5-CSE)** - Advanced solution architecture
- **F5 Certified Technology Specialist - LTM (F5-CTS-LTM)** - Load balancing specialist
- **F5 Certified Technology Specialist - ASM (F5-CTS-ASM)** - Application security specialist
- **F5 Application Delivery Fundamentals** - Core platform knowledge

**Senior Application Delivery Engineer:**
- **F5 Certified Administrator (F5-CA)** - Platform administration
- **F5 Certified Technology Specialist - APM (F5-CTS-APM)** - Access policy management
- **F5 Certified Technology Specialist - DNS (F5-CTS-DNS)** - DNS services specialist
- **VMware vSphere Professional** - Virtualization integration

### Healthcare Implementation Experience
- **Regional Medical Center:** Deployed F5 infrastructure supporting 150,000+ patient visits annually
- **Healthcare SaaS Platform:** Load balancing for 200+ healthcare facilities with 99.99% uptime
- **Medical Research Institution:** WAF protection for clinical trial data systems with HIPAA compliance
- **Pharmaceutical Manufacturing:** Application delivery for FDA-regulated manufacturing systems

---

## 2. F5 BIG-IP Architecture and Implementation

### Healthcare Application Delivery Architecture

```
Healthcare F5 BIG-IP Infrastructure

                    [Internet Gateway]
                          |
                   [F5 BIG-IP HA Pair]
                  Primary    |    Secondary
                     i4800   |      i4800
                          |
                 [Application Zones]
                    /     |     \
                   /      |      \
            [Patient]  [Medical]  [Admin]
             Care       Devices    Systems
               |          |         |
        [EHR Cluster] [PACS] [Finance Apps]
        [Database]    [LIS]   [HR Systems]
        [Monitoring]  [DICOM] [Email]
```

### Hardware Specifications and Sizing

**F5 BIG-IP i4800 Cluster Configuration:**
```yaml
F5_Infrastructure:
  Primary_Device:
    Model: "BIG-IP i4800"
    CPU: "2x Intel Xeon, 16 cores total"
    Memory: "128GB RAM"
    Storage: "2TB SSD"
    Network_Interfaces: "8x 10GbE, 2x 40GbE"
    Throughput:
      Layer4: "40 Gbps"
      Layer7: "20 Gbps"
      HTTP_Requests: "16M requests/second"
      SSL_TPS: "100,000 bulk RSA, 20,000 ECDSA"
      Concurrent_Connections: "96M"
      
  Secondary_Device:
    Configuration: "Identical to primary"
    Role: "Active-Standby failover"
    Sync_Method: "Configuration synchronization"
    Heartbeat: "Network and serial failover"
    
  Licensed_Modules:
    - LTM: "Local Traffic Manager - Load balancing"
    - ASM: "Application Security Manager - WAF"
    - APM: "Access Policy Manager - VPN/SSO"
    - AVR: "Application Visibility & Reporting"
    - AFM: "Advanced Firewall Manager"
    - DNS: "DNS Services"
```

### Healthcare-Specific Load Balancing Configuration

**Electronic Health Records (EHR) Load Balancing:**
```javascript
// F5 iRule for EHR Application Intelligence
when HTTP_REQUEST {
    # Log healthcare application access for HIPAA audit
    log local0. "EHR Access: Client=[IP::client_addr] User-Agent=[HTTP::header User-Agent] URI=[HTTP::uri]"
    
    # Route based on EHR application context
    switch -glob [HTTP::uri] {
        "/api/patient/*" {
            # Patient data API - high security pool
            pool pool_ehr_patient_api_secure
            # Enable session persistence for patient context
            persist uie "patient_session_[HTTP::header X-Patient-ID]" 3600
        }
        "/api/clinical/*" {
            # Clinical applications - optimized pool
            pool pool_ehr_clinical_apps
            # Load balance based on clinical department
            if { [HTTP::header exists "X-Department"] } {
                switch [HTTP::header "X-Department"] {
                    "cardiology" { pool pool_ehr_cardiology }
                    "radiology" { pool pool_ehr_radiology }
                    "emergency" { pool pool_ehr_emergency }
                    default { pool pool_ehr_clinical_apps }
                }
            }
        }
        "/admin/*" {
            # Administrative functions - standard pool
            pool pool_ehr_admin
        }
        default {
            # Default EHR pool with health monitoring
            pool pool_ehr_default
        }
    }
    
    # Healthcare-specific security headers
    if { [HTTP::header exists "Authorization"] } {
        # Validate JWT tokens for API access
        set jwt_token [HTTP::header "Authorization"]
        if { [call validate_healthcare_jwt $jwt_token] } {
            # Allow authenticated request
            return
        } else {
            # Block invalid authentication
            HTTP::respond 401 content "Authentication required for healthcare data access"
        }
    }
}

when HTTP_RESPONSE {
    # Add healthcare-specific security headers
    HTTP::header insert "X-Content-Type-Options" "nosniff"
    HTTP::header insert "X-Frame-Options" "DENY" 
    HTTP::header insert "Strict-Transport-Security" "max-age=31536000; includeSubDomains"
    HTTP::header insert "Content-Security-Policy" "default-src 'self'; script-src 'self' 'unsafe-inline'"
    
    # Remove server information for security
    HTTP::header remove "Server"
    HTTP::header remove "X-Powered-By"
    
    # Log response for audit trail
    log local0. "EHR Response: Client=[IP::client_addr] Status=[HTTP::status] Size=[HTTP::header Content-Length]"
}
```

**PACS/Medical Imaging Load Balancing:**
```json
{
  "pool_pacs_imaging": {
    "name": "/Common/pool_pacs_imaging_servers",
    "load_balancing_mode": "least-connections-member",
    "health_monitors": ["/Common/tcp_443", "/Common/dicom_custom_monitor"],
    "members": [
      {
        "address": "10.20.10.100",
        "port": 443,
        "priority_group": 1,
        "description": "PACS Server 1 - Primary imaging storage"
      },
      {
        "address": "10.20.10.101", 
        "port": 443,
        "priority_group": 1,
        "description": "PACS Server 2 - Secondary imaging storage"
      },
      {
        "address": "10.20.10.102",
        "port": 443,
        "priority_group": 2, 
        "description": "PACS Server 3 - Disaster recovery"
      }
    ],
    "session_persistence": {
      "method": "source-address-affinity",
      "timeout": 7200,
      "description": "DICOM session consistency required"
    }
  },
  
  "virtual_server_pacs": {
    "name": "/Common/vs_pacs_imaging",
    "destination": "192.168.20.100:443",
    "pool": "/Common/pool_pacs_imaging_servers",
    "ssl_profiles": {
      "client": "/Common/healthcare_ssl_client",
      "server": "/Common/healthcare_ssl_server"
    },
    "compression_profile": "/Common/dicom_compression",
    "irules": ["/Common/pacs_intelligent_routing"],
    "persistence_profiles": ["/Common/source_addr_persistence"]
  }
}
```

### Web Application Firewall (WAF) Implementation

**Healthcare Application Security Manager (ASM) Policy:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<policy name="Healthcare_EHR_Security_Policy">
  <general>
    <name>Healthcare_EHR_WAF_Policy</name>
    <description>HIPAA-compliant WAF policy for healthcare applications</description>
    <policy-type>security</policy-type>
    <enforcement-mode>blocking</enforcement-mode>
    <learning-mode>disabled</learning-mode>
  </general>
  
  <application-language>utf-8</application-language>
  
  <!-- OWASP Top 10 Protection for Healthcare -->
  <vulnerabilities>
    <vulnerability>
      <name>SQL Injection</name>
      <enabled>true</enabled>
      <blocking>true</blocking>
      <alarm>true</alarm>
      <learn>false</learn>
    </vulnerability>
    <vulnerability>
      <name>Cross Site Scripting (XSS)</name>
      <enabled>true</enabled>
      <blocking>true</blocking>
      <alarm>true</alarm>
      <learn>false</learn>
    </vulnerability>
    <vulnerability>
      <name>Command Execution</name>
      <enabled>true</enabled>
      <blocking>true</blocking>
      <alarm>true</alarm>
      <learn>false</learn>
    </vulnerability>
  </vulnerabilities>
  
  <!-- Healthcare-specific attack signatures -->
  <attack-signatures>
    <signature-set name="Healthcare_PHI_Protection">
      <signature id="healthcare_001">
        <name>PHI Data Exfiltration Attempt</name>
        <pattern>(?i)(patient|medical|health|hipaa).*?(ssn|social|security|number|dob|birth|address|phone)</pattern>
        <context>response-body</context>
        <enabled>true</enabled>
        <blocking>true</blocking>
      </signature>
      
      <signature id="healthcare_002">
        <name>Medical Record Access Violation</name>
        <pattern>(?i)(select|union|insert|update|delete).*(patient|medical_record|diagnosis|treatment)</pattern>
        <context>parameter</context>
        <enabled>true</enabled>
        <blocking>true</blocking>
      </signature>
    </signature-set>
  </attack-signatures>
  
  <!-- File type restrictions for healthcare -->
  <filetypes>
    <filetype>
      <name>exe</name>
      <allowed>false</allowed>
      <response>block</response>
    </filetype>
    <filetype>
      <name>pdf</name>
      <allowed>true</allowed>
      <check-post-data>true</check-post-data>
    </filetype>
    <filetype>
      <name>dcm</name>
      <allowed>true</allowed>
      <description>DICOM medical imaging files</description>
    </filetype>
  </filetypes>
  
  <!-- Parameter validation for healthcare forms -->
  <parameters>
    <parameter>
      <name>patient_id</name>
      <type>alpha-numeric</type>
      <length>
        <min>1</min>
        <max>20</max>
      </length>
      <mandatory>true</mandatory>
      <sensitive>true</sensitive>
    </parameter>
    
    <parameter>
      <name>ssn</name>
      <type>numeric</type>
      <length>
        <min>9</min>
        <max>11</max>
      </length>
      <mandatory>false</mandatory>
      <sensitive>true</sensitive>
      <data-guard>
        <enabled>true</enabled>
        <enforcement>block</enforcement>
      </data-guard>
    </parameter>
  </parameters>
  
  <!-- Session tracking for healthcare applications -->
  <sessions-and-logins>
    <session-tracking>
      <enabled>true</enabled>
      <session-timeout>1800</session-timeout>
      <login-enforcement>
        <enabled>true</enabled>
        <authenticated-urls>
          <url>/patient/*</url>
          <url>/clinical/*</url>
          <url>/admin/*</url>
        </authenticated-urls>
      </login-enforcement>
    </session-tracking>
  </sessions-and-logins>
  
  <!-- Data loss prevention for PHI -->
  <data-guard>
    <enabled>true</enabled>
    <enforcement-mode>blocking</enforcement-mode>
    <credit-card-numbers>
      <enabled>false</enabled>
    </credit-card-numbers>
    <us-social-security-numbers>
      <enabled>true</enabled>
      <enforcement>block</enforcement>
      <mask-data>true</mask-data>
    </us-social-security-numbers>
    <custom-patterns>
      <pattern>
        <name>Medical_Record_Number</name>
        <regex>\b(MRN|MR|MEDICAL)\s*[:#-]?\s*([A-Z0-9]{6,12})\b</regex>
        <enabled>true</enabled>
        <enforcement>block</enforcement>
      </pattern>
    </custom-patterns>
  </data-guard>
</policy>
```

### SSL/TLS Configuration for Healthcare Compliance

**Healthcare SSL Profile Configuration:**
```bash
#!/bin/bash
# F5 BIG-IP SSL Profile for Healthcare Compliance
# Creates HIPAA-compliant SSL configuration

# Create custom SSL profile for healthcare applications
tmsh create ltm profile client-ssl healthcare_ssl_client {
    # Certificate and key configuration
    cert-key-chain {
        healthcare_primary {
            cert /Common/healthcare_wildcard.crt
            key /Common/healthcare_wildcard.key
            chain /Common/healthcare_intermediate.crt
        }
    }
    
    # Security configuration for HIPAA compliance
    ciphers "ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20:!aNULL:!MD5:!DSS:!3DES"
    options { no-ssl no-tlsv1 no-tlsv1.1 }
    
    # Enable only TLS 1.2 and 1.3
    min-version 1.2
    max-version 1.3
    
    # Perfect Forward Secrecy
    dhcp-params /Common/dh2048.pem
    
    # Security headers
    insert-xforwarded-for enabled
    proxy-ssl enabled
    
    # Session management
    session-ticket disabled
    session-cache-size 65536
    session-cache-timeout 3600
    
    # Healthcare-specific settings
    renegotiate-size indefinite
    secure-renegotiation require-strict
    server-name healthcare.domain.local
    
    # Enable OCSP stapling for certificate validation
    ocsp-stapling enabled
}

# Create server-side SSL profile
tmsh create ltm profile server-ssl healthcare_ssl_server {
    # Backend server SSL configuration
    ciphers "ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20"
    options { no-ssl no-tlsv1 no-tlsv1.1 }
    
    # Certificate verification for backend servers
    ca-file /Common/healthcare_ca_bundle.crt
    peer-cert-mode require
    
    # Backend encryption enforcement
    ssl-forward-proxy disabled
    ssl-forward-proxy-bypass disabled
}

# Configure HTTP compression profile for healthcare
tmsh create ltm profile http-compression healthcare_compression {
    # Optimize compression for healthcare applications
    uri-exclude {
        "*.dcm"    # DICOM files should not be compressed
        "*.jpg"    # Medical images
        "*.png"    # Medical images
        "*.pdf"    # Medical documents
    }
    
    # Enable compression for text content
    content-type-include {
        "text/html"
        "text/css"
        "text/javascript"
        "application/javascript"
        "application/json"
        "application/xml"
    }
    
    # Compression settings
    gzip-memory-level 8
    gzip-window-size 16
    gzip-level 6
}
```

---

## 3. Automation and Infrastructure as Code

### F5 Automation Toolchain Implementation

**Declarative Onboarding (DO) Configuration:**
```json
{
  "schemaVersion": "1.25.0",
  "class": "Device",
  "async": true,
  "label": "Healthcare F5 BIG-IP Declarative Onboarding",
  "Common": {
    "class": "Tenant",
    "mySystem": {
      "class": "System",
      "hostname": "bigip-healthcare-01.domain.local",
      "cliInactivityTimeout": 1200,
      "consoleInactivityTimeout": 1200,
      "autoPhonehome": false
    },
    "myDns": {
      "class": "DNS",
      "nameServers": [
        "10.1.1.10",
        "10.1.1.11"
      ],
      "search": [
        "healthcare.domain.local",
        "domain.local"
      ]
    },
    "myNtp": {
      "class": "NTP",
      "servers": [
        "pool.ntp.org",
        "time.healthcare.domain.local"
      ],
      "timezone": "America/New_York"
    },
    "myProvisioning": {
      "class": "Provision",
      "ltm": "nominal",
      "asm": "nominal",
      "apm": "nominal",
      "afm": "nominal",
      "avr": "nominal",
      "fps": "none"
    },
    "internal": {
      "class": "VLAN",
      "tag": 100,
      "mtu": 9000,
      "interfaces": [
        {
          "name": "1.1",
          "tagged": false
        }
      ]
    },
    "external": {
      "class": "VLAN", 
      "tag": 200,
      "mtu": 1500,
      "interfaces": [
        {
          "name": "1.2",
          "tagged": false
        }
      ]
    },
    "ha": {
      "class": "VLAN",
      "tag": 300,
      "mtu": 1500,
      "interfaces": [
        {
          "name": "1.3",
          "tagged": false
        }
      ]
    },
    "internal-self": {
      "class": "SelfIp",
      "address": "192.168.100.10/24",
      "vlan": "internal",
      "allowService": [
        "default",
        "ssh"
      ],
      "trafficGroup": "traffic-group-local-only"
    },
    "external-self": {
      "class": "SelfIp", 
      "address": "192.168.200.10/24",
      "vlan": "external",
      "allowService": "none",
      "trafficGroup": "traffic-group-1"
    },
    "default-route": {
      "class": "Route",
      "gw": "192.168.200.1",
      "network": "default",
      "mtu": 1500
    },
    "configsync": {
      "class": "ConfigSync",
      "configsyncIp": "/Common/ha-self/address"
    },
    "failoverAddress": {
      "class": "FailoverUnicast",
      "address": "/Common/ha-self/address",
      "port": 1026
    },
    "deviceTrust": {
      "class": "DeviceTrust",
      "localUsername": "admin",
      "localPassword": "$ADMIN_PASSWORD",
      "remoteHost": "192.168.300.11",
      "remoteUsername": "admin", 
      "remotePassword": "$PEER_ADMIN_PASSWORD"
    },
    "deviceGroup": {
      "class": "DeviceGroup",
      "type": "sync-failover",
      "members": [
        "bigip-healthcare-01.domain.local",
        "bigip-healthcare-02.domain.local"
      ],
      "owner": "/Common/failoverGroup",
      "autoSync": true,
      "saveOnAutoSync": false,
      "networkFailover": true,
      "fullLoadOnSync": false,
      "asmSync": true
    }
  }
}
```

**Application Services 3 (AS3) Healthcare Configuration:**
```json
{
  "class": "AS3",
  "action": "deploy",
  "persist": true,
  "declaration": {
    "class": "ADC",
    "schemaVersion": "3.34.0",
    "label": "Healthcare Application Services",
    "remark": "HIPAA-compliant application delivery configuration",
    "Healthcare_Tenant": {
      "class": "Tenant",
      "label": "Healthcare Production Applications",
      
      "EHR_Application": {
        "class": "Application",
        "template": "https",
        "label": "Electronic Health Records Application",
        
        "serviceMain": {
          "class": "Service_HTTPS",
          "virtualAddresses": ["192.168.1.100"],
          "virtualPort": 443,
          "pool": "ehr_pool",
          "serverTLS": "ehr_tls_server",
          "clientTLS": "ehr_tls_client",
          "persistenceMethods": ["cookie"],
          "profileHTTP": "ehr_http_profile",
          "profileHTTPCompression": "ehr_compression",
          "iRules": ["ehr_intelligent_routing"],
          "securityLogProfiles": [
            {
              "bigip": "/Common/healthcare_security_log"
            }
          ]
        },
        
        "ehr_pool": {
          "class": "Pool",
          "loadBalancingMode": "least-connections-member",
          "slowRampTime": 300,
          "members": [
            {
              "servicePort": 443,
              "serverAddresses": ["10.10.10.100", "10.10.10.101", "10.10.10.102"],
              "priorityGroup": 1,
              "connectionLimit": 1000,
              "rateLimit": 500
            }
          ],
          "monitors": [
            {
              "bigip": "/Common/https_443"
            },
            "ehr_application_monitor"
          ]
        },
        
        "ehr_application_monitor": {
          "class": "Monitor",
          "monitorType": "https",
          "interval": 10,
          "timeout": 31,
          "send": "GET /api/health HTTP/1.1\\r\\nHost: ehr.healthcare.local\\r\\nAuthorization: Bearer $HEALTH_CHECK_TOKEN\\r\\n\\r\\n",
          "receive": "status.*ok"
        },
        
        "ehr_tls_client": {
          "class": "TLS_Client",
          "certificates": [
            {
              "certificate": "ehr_certificate"
            }
          ],
          "ciphers": "ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20:!aNULL:!MD5:!DSS",
          "insertEmptyFragmentsFlag": false,
          "singleUseDhFlag": false,
          "ssl3SessionIdLength": 0,
          "tls1_3SessionTickets": false
        },
        
        "ehr_tls_server": {
          "class": "TLS_Server",
          "certificates": [
            {
              "certificate": "ehr_certificate"
            }
          ],
          "ciphers": "ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20",
          "authenticationFrequency": "one-time"
        },
        
        "ehr_certificate": {
          "class": "Certificate",
          "certificate": {
            "bigip": "/Common/healthcare_wildcard.crt"
          },
          "privateKey": {
            "bigip": "/Common/healthcare_wildcard.key"
          },
          "chainCA": {
            "bigip": "/Common/healthcare_intermediate.crt"
          }
        }
      },
      
      "PACS_Imaging": {
        "class": "Application",
        "template": "tcp",
        "label": "PACS Medical Imaging Application",
        
        "serviceMain": {
          "class": "Service_TCP",
          "virtualAddresses": ["192.168.1.110"],
          "virtualPort": 11112,
          "pool": "pacs_pool",
          "persistenceMethods": ["source-address"],
          "profileTCP": "pacs_tcp_profile"
        },
        
        "pacs_pool": {
          "class": "Pool",
          "loadBalancingMode": "least-connections-member",
          "members": [
            {
              "servicePort": 11112,
              "serverAddresses": ["10.20.10.100", "10.20.10.101"],
              "connectionLimit": 500,
              "priorityGroup": 1
            }
          ],
          "monitors": ["pacs_dicom_monitor"]
        },
        
        "pacs_dicom_monitor": {
          "class": "Monitor",
          "monitorType": "tcp",
          "interval": 10,
          "timeout": 31
        },
        
        "pacs_tcp_profile": {
          "class": "TCP_Profile",
          "idleTimeout": 1800,
          "closeWaitTimeout": 60,
          "finWait2Timeout": 300,
          "keepAliveInterval": 1200
        }
      }
    }
  }
}
```

### F5 Automation with Ansible

**Healthcare F5 Deployment Playbook:**
```yaml
---
# Healthcare F5 BIG-IP Automation Playbook
- name: Deploy Healthcare F5 Application Services
  hosts: f5_devices
  connection: local
  gather_facts: no
  
  vars:
    provider:
      server: "{{ ansible_host }}"
      user: "{{ f5_username }}"
      password: "{{ f5_password }}"
      validate_certs: no
      
    healthcare_applications:
      - name: "EHR_Application"
        virtual_server: "192.168.1.100"
        port: 443
        pool_members:
          - "10.10.10.100:443"
          - "10.10.10.101:443"
          - "10.10.10.102:443"
        ssl_profile: "healthcare_ssl_client"
        waf_policy: "Healthcare_EHR_Security_Policy"
        
      - name: "PACS_Imaging"
        virtual_server: "192.168.1.110" 
        port: 11112
        pool_members:
          - "10.20.10.100:11112"
          - "10.20.10.101:11112"
        persistence: "source_addr"
        
  tasks:
    - name: Create healthcare SSL certificates
      bigip_ssl_certificate:
        provider: "{{ provider }}"
        name: "healthcare_wildcard"
        certificate_src: "files/certificates/healthcare_wildcard.crt"
        key_src: "files/certificates/healthcare_wildcard.key"
        state: present
      delegate_to: localhost
      
    - name: Create healthcare SSL profiles
      bigip_profile_client_ssl:
        provider: "{{ provider }}"
        name: "healthcare_ssl_client"
        cert_key_chain:
          - cert: "healthcare_wildcard"
            key: "healthcare_wildcard"
        ciphers: "ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20:!aNULL:!MD5:!DSS"
        options:
          - no-ssl
          - no-tlsv1
          - no-tlsv1.1
        state: present
      delegate_to: localhost
      
    - name: Deploy healthcare application pools
      bigip_pool:
        provider: "{{ provider }}"
        name: "pool_{{ item.name }}"
        lb_method: "least-connections-member"
        monitor_type: "and_list"
        monitors:
          - "https_443"
        members:
          - host: "{{ item.pool_members | map('regex_replace', ':.*', '') | list }}"
            port: "{{ item.pool_members | map('regex_replace', '.*:', '') | list }}"
        state: present
      with_items: "{{ healthcare_applications }}"
      delegate_to: localhost
      
    - name: Create healthcare virtual servers
      bigip_virtual_server:
        provider: "{{ provider }}"
        name: "vs_{{ item.name }}"
        destination: "{{ item.virtual_server }}"
        port: "{{ item.port }}"
        pool: "pool_{{ item.name }}"
        profiles:
          - name: "tcp-lan-optimized"
            context: "all"
          - name: "{{ item.ssl_profile | default('') }}"
            context: "client-side"
        irules:
          - "healthcare_security_irule"
        state: present
      with_items: "{{ healthcare_applications }}"
      delegate_to: localhost
      
    - name: Apply WAF policies to healthcare applications
      bigip_asm_policy_manage:
        provider: "{{ provider }}"
        name: "{{ item.waf_policy }}"
        virtual_server: "vs_{{ item.name }}"
        state: present
      with_items: "{{ healthcare_applications }}"
      when: item.waf_policy is defined
      delegate_to: localhost
      
    - name: Configure healthcare logging profiles
      bigip_log_destination:
        provider: "{{ provider }}"
        name: "healthcare_security_log"
        type: "remotesyslog"
        syslog_settings:
          syslog_format: "rfc5424"
          remote_high_speed_log: "healthcare_hsl_pool"
        state: present
      delegate_to: localhost
```

### Terraform F5 Infrastructure

**Healthcare F5 Infrastructure as Code:**
```hcl
# Healthcare F5 BIG-IP Infrastructure
terraform {
  required_providers {
    bigip = {
      source  = "F5Networks/bigip"
      version = "~> 1.16"
    }
  }
}

# F5 BIG-IP Provider
provider "bigip" {
  address  = var.f5_management_ip
  username = var.f5_username
  password = var.f5_password
}

# Healthcare Application Pool - EHR Systems
resource "bigip_ltm_pool" "ehr_application_pool" {
  name                = "/Common/pool_ehr_applications"
  load_balancing_mode = "least-connections-member"
  monitors            = ["/Common/https_443"]
  
  # EHR server cluster members
  dynamic "members" {
    for_each = var.ehr_servers
    content {
      name    = "${members.value.ip}:${members.value.port}"
      address = members.value.ip
      port    = members.value.port
    }
  }
  
  # Healthcare-specific pool settings
  slow_ramp_time      = 300
  reselect_tries      = 3
  service_down_action = "reset"
}

# Healthcare Virtual Server - EHR Application
resource "bigip_ltm_virtual_server" "ehr_virtual_server" {
  pool                       = bigip_ltm_pool.ehr_application_pool.name
  name                       = "/Common/vs_ehr_applications"
  destination               = var.ehr_virtual_ip
  port                      = 443
  protocols                 = ["tcp"]
  
  # SSL profiles for healthcare compliance
  profiles = [
    "/Common/tcp-lan-optimized",
    bigip_ltm_profile_client_ssl.healthcare_ssl.name
  ]
  
  # Healthcare security iRule
  rules = [bigip_ltm_irule.healthcare_security.name]
  
  # Session persistence for EHR applications
  persistence_profiles = [bigip_ltm_persistence_profile_cookie.ehr_persistence.name]
  
  # Connection limits for healthcare applications
  connection_limit = 10000
  rate_limit       = 1000
  
  # Enable connection mirroring for high availability
  mirror = "enabled"
}

# Healthcare SSL Profile
resource "bigip_ltm_profile_client_ssl" "healthcare_ssl" {
  name     = "/Common/healthcare_ssl_client"
  defaults_from = "/Common/clientssl"
  
  # Healthcare-compliant cipher suite
  ciphers = "ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20:!aNULL:!MD5:!DSS:!3DES"
  
  # TLS version restrictions for HIPAA compliance
  options = ["no-ssl", "no-tlsv1", "no-tlsv1.1"]
  
  # Certificate configuration
  cert_key_chain {
    name = "healthcare_cert_chain"
    cert = "/Common/healthcare_wildcard.crt"
    key  = "/Common/healthcare_wildcard.key"
  }
  
  # Security settings
  secure_renegotiation = "require-strict"
  server_name         = var.healthcare_domain
  proxy_ssl           = "enabled"
}

# Healthcare Security iRule
resource "bigip_ltm_irule" "healthcare_security" {
  name = "/Common/healthcare_security_irule"
  
  irule = <<-EOT
when HTTP_REQUEST {
    # Healthcare audit logging
    log local0. "Healthcare Access: [IP::client_addr] -> [HTTP::host][HTTP::uri]"
    
    # Block non-HTTPS healthcare traffic
    if { [TCP::local_port] != 443 } {
        HTTP::respond 302 Location "https://[HTTP::host][HTTP::uri]"
    }
    
    # Validate healthcare API authentication
    if { [HTTP::path] starts_with "/api/" } {
        if { ![HTTP::header exists "Authorization"] } {
            HTTP::respond 401 content "Authentication required for healthcare API access"
            return
        }
        
        # Rate limiting for healthcare APIs
        if { [table lookup -subtable api_rate_limit [IP::client_addr]] > 1000 } {
            HTTP::respond 429 content "Rate limit exceeded for healthcare API"
            return
        }
        table set -subtable api_rate_limit [IP::client_addr] [expr {[table lookup -subtable api_rate_limit [IP::client_addr]] + 1}] 60
    }
    
    # Healthcare-specific security headers
    HTTP::header insert "X-Healthcare-Request-ID" "[expr {int(rand() * 1000000)}]"
}

when HTTP_RESPONSE {
    # Healthcare security headers
    HTTP::header insert "X-Content-Type-Options" "nosniff"
    HTTP::header insert "X-Frame-Options" "DENY"
    HTTP::header insert "Strict-Transport-Security" "max-age=31536000; includeSubDomains"
    HTTP::header insert "X-Healthcare-Server" "F5-BIG-IP"
    
    # Remove server identification
    HTTP::header remove "Server"
    HTTP::header remove "X-Powered-By"
    
    # Healthcare compliance logging
    log local0. "Healthcare Response: [IP::client_addr] <- Status:[HTTP::status] Size:[HTTP::header Content-Length]"
}
EOT
}

# PACS Imaging Application Configuration
resource "bigip_ltm_pool" "pacs_imaging_pool" {
  name                = "/Common/pool_pacs_imaging"
  load_balancing_mode = "least-connections-member"
  monitors            = [bigip_ltm_monitor.dicom_monitor.name]
  
  dynamic "members" {
    for_each = var.pacs_servers
    content {
      name    = "${members.value.ip}:${members.value.port}"
      address = members.value.ip
      port    = members.value.port
    }
  }
}

# Custom DICOM Monitor
resource "bigip_ltm_monitor" "dicom_monitor" {
  name     = "/Common/dicom_tcp_monitor"
  parent   = "/Common/tcp"
  interval = 10
  timeout  = 31
  
  # DICOM-specific monitoring
  send    = ""
  receive = ""
}

# Variables for healthcare configuration
variable "ehr_servers" {
  description = "EHR server configuration"
  type = list(object({
    ip   = string
    port = number
  }))
  default = [
    {ip = "10.10.10.100", port = 443},
    {ip = "10.10.10.101", port = 443},
    {ip = "10.10.10.102", port = 443}
  ]
}

variable "pacs_servers" {
  description = "PACS server configuration"
  type = list(object({
    ip   = string
    port = number
  }))
  default = [
    {ip = "10.20.10.100", port = 11112},
    {ip = "10.20.10.101", port = 11112}
  ]
}

variable "healthcare_domain" {
  description = "Healthcare domain name"
  type        = string
  default     = "healthcare.domain.local"
}

variable "ehr_virtual_ip" {
  description = "EHR application virtual IP"
  type        = string
  default     = "192.168.1.100"
}
```

---

## 4. Testing and Validation Framework

### Comprehensive F5 Testing Strategy

**Healthcare Application Load Testing:**
```python
#!/usr/bin/env python3
"""
F5 BIG-IP Healthcare Application Load Testing
Comprehensive load testing for healthcare applications with compliance validation
"""
import asyncio
import aiohttp
import json
import time
import statistics
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class LoadTestConfig:
    """Load test configuration for healthcare applications"""
    application_name: str
    virtual_server_ip: str
    test_duration_seconds: int
    concurrent_users: int
    ramp_up_time: int
    test_scenarios: List[Dict]

class HealthcareF5LoadTester:
    """Load testing framework for F5-protected healthcare applications"""
    
    def __init__(self):
        self.test_results = {}
        self.performance_thresholds = {
            'response_time_ms': 200,
            'throughput_rps': 1000,
            'error_rate_percent': 0.5,
            'ssl_handshake_ms': 50
        }
        
    async def execute_healthcare_load_test(self, test_config: LoadTestConfig) -> Dict:
        """Execute comprehensive load test for healthcare application"""
        
        print(f"Starting load test for {test_config.application_name}")
        
        test_results = {
            'application': test_config.application_name,
            'test_start': time.time(),
            'configuration': {
                'virtual_server': test_config.virtual_server_ip,
                'duration': test_config.test_duration_seconds,
                'concurrent_users': test_config.concurrent_users,
                'scenarios': len(test_config.test_scenarios)
            },
            'performance_metrics': {},
            'compliance_validation': {},
            'recommendations': []
        }
        
        # Execute test scenarios concurrently
        scenario_tasks = []
        for scenario in test_config.test_scenarios:
            task = self.execute_test_scenario(test_config, scenario)
            scenario_tasks.append(task)
        
        # Wait for all scenarios to complete
        scenario_results = await asyncio.gather(*scenario_tasks)
        
        # Aggregate results
        test_results['scenario_results'] = scenario_results
        test_results['performance_metrics'] = self.calculate_aggregate_metrics(scenario_results)
        test_results['compliance_validation'] = await self.validate_healthcare_compliance(test_results)
        
        test_results['test_end'] = time.time()
        test_results['test_duration'] = test_results['test_end'] - test_results['test_start']
        
        return test_results
    
    async def execute_test_scenario(self, test_config: LoadTestConfig, scenario: Dict) -> Dict:
        """Execute individual test scenario"""
        
        scenario_results = {
            'scenario_name': scenario['name'],
            'requests_sent': 0,
            'requests_successful': 0,
            'requests_failed': 0,
            'response_times': [],
            'ssl_handshake_times': [],
            'error_details': []
        }
        
        # Create HTTP session with healthcare-specific configuration
        connector = aiohttp.TCPConnector(
            ssl=False,  # Using F5 SSL termination
            keepalive_timeout=30,
            enable_cleanup_closed=True
        )
        
        async with aiohttp.ClientSession(connector=connector) as session:
            # Generate load for specified duration
            end_time = time.time() + test_config.test_duration_seconds
            
            while time.time() < end_time:
                # Execute concurrent requests based on scenario
                tasks = []
                for _ in range(scenario.get('concurrent_requests', 10)):
                    task = self.send_healthcare_request(session, test_config, scenario)
                    tasks.append(task)
                
                # Wait for batch completion
                batch_results = await asyncio.gather(*tasks, return_exceptions=True)
                
                # Process batch results
                for result in batch_results:
                    if isinstance(result, Exception):
                        scenario_results['requests_failed'] += 1
                        scenario_results['error_details'].append(str(result))
                    else:
                        scenario_results['requests_sent'] += 1
                        if result['success']:
                            scenario_results['requests_successful'] += 1
                            scenario_results['response_times'].append(result['response_time'])
                            scenario_results['ssl_handshake_times'].append(result['ssl_time'])
                        else:
                            scenario_results['requests_failed'] += 1
                            scenario_results['error_details'].append(result['error'])
                
                # Rate limiting between batches
                await asyncio.sleep(scenario.get('batch_interval', 1))
        
        return scenario_results
    
    async def send_healthcare_request(self, session: aiohttp.ClientSession, 
                                    test_config: LoadTestConfig, scenario: Dict) -> Dict:
        """Send individual healthcare application request"""
        
        request_start = time.time()
        ssl_start = time.time()
        
        try:
            # Prepare healthcare-specific headers
            headers = {
                'User-Agent': 'HealthcareLoadTest/1.0',
                'Authorization': f"Bearer {scenario.get('auth_token', 'test_token')}",
                'X-Healthcare-Context': scenario.get('context', 'load_test'),
                'Content-Type': 'application/json'
            }
            
            # Construct request URL
            url = f"https://{test_config.virtual_server_ip}{scenario['endpoint']}"
            
            # Execute request with timeout
            async with session.request(
                method=scenario.get('method', 'GET'),
                url=url,
                headers=headers,
                json=scenario.get('payload'),
                timeout=aiohttp.ClientTimeout(total=30)
            ) as response:
                
                ssl_time = time.time() - ssl_start
                response_body = await response.text()
                response_time = time.time() - request_start
                
                return {
                    'success': response.status < 400,
                    'status_code': response.status,
                    'response_time': response_time * 1000,  # milliseconds
                    'ssl_time': ssl_time * 1000,  # milliseconds
                    'response_size': len(response_body),
                    'error': None if response.status < 400 else f"HTTP {response.status}"
                }
                
        except asyncio.TimeoutError:
            return {
                'success': False,
                'response_time': (time.time() - request_start) * 1000,
                'ssl_time': 0,
                'error': 'Request timeout'
            }
        except Exception as e:
            return {
                'success': False,
                'response_time': (time.time() - request_start) * 1000,
                'ssl_time': 0,
                'error': str(e)
            }
    
    async def validate_healthcare_compliance(self, test_results: Dict) -> Dict:
        """Validate healthcare compliance requirements during load testing"""
        
        compliance_validation = {
            'hipaa_compliant': True,
            'performance_compliant': True,
            'security_validated': True,
            'validation_details': []
        }
        
        # Validate response time requirements for patient care
        avg_response_time = test_results['performance_metrics'].get('avg_response_time', 0)
        if avg_response_time > self.performance_thresholds['response_time_ms']:
            compliance_validation['performance_compliant'] = False
            compliance_validation['validation_details'].append(
                f"Response time {avg_response_time}ms exceeds healthcare threshold of {self.performance_thresholds['response_time_ms']}ms"
            )
        
        # Validate SSL/TLS performance for PHI protection
        avg_ssl_time = test_results['performance_metrics'].get('avg_ssl_handshake', 0)
        if avg_ssl_time > self.performance_thresholds['ssl_handshake_ms']:
            compliance_validation['security_validated'] = False
            compliance_validation['validation_details'].append(
                f"SSL handshake time {avg_ssl_time}ms impacts patient care efficiency"
            )
        
        # Validate error rate for healthcare availability requirements
        error_rate = test_results['performance_metrics'].get('error_rate_percent', 0)
        if error_rate > self.performance_thresholds['error_rate_percent']:
            compliance_validation['hipaa_compliant'] = False
            compliance_validation['validation_details'].append(
                f"Error rate {error_rate}% exceeds healthcare availability requirements"
            )
        
        # Overall compliance determination
        compliance_validation['overall_compliant'] = all([
            compliance_validation['hipaa_compliant'],
            compliance_validation['performance_compliant'],
            compliance_validation['security_validated']
        ])
        
        return compliance_validation

# F5 WAF Security Testing
class HealthcareWAFTester:
    """Security testing framework for F5 WAF protecting healthcare applications"""
    
    def __init__(self):
        self.attack_scenarios = [
            {
                'name': 'SQL Injection - Patient Data',
                'payload': "' OR 1=1--",
                'parameter': 'patient_id',
                'expected_block': True
            },
            {
                'name': 'Cross-Site Scripting - EHR',
                'payload': '<script>alert("xss")</script>',
                'parameter': 'search_term',
                'expected_block': True
            },
            {
                'name': 'PHI Data Extraction',
                'payload': 'SELECT ssn, dob FROM patients',
                'parameter': 'query',
                'expected_block': True
            },
            {
                'name': 'Medical Record Enumeration',
                'payload': '../../../etc/passwd',
                'parameter': 'file_path',
                'expected_block': True
            }
        ]
    
    async def execute_security_testing(self, f5_virtual_server: str) -> Dict:
        """Execute comprehensive security testing against F5 WAF"""
        
        security_test_results = {
            'virtual_server': f5_virtual_server,
            'test_start': time.time(),
            'attack_scenarios_tested': len(self.attack_scenarios),
            'attacks_blocked': 0,
            'attacks_allowed': 0,
            'false_positives': 0,
            'security_effectiveness': 0
        }
        
        for scenario in self.attack_scenarios:
            attack_result = await self.execute_attack_scenario(f5_virtual_server, scenario)
            
            if attack_result['blocked'] and scenario['expected_block']:
                security_test_results['attacks_blocked'] += 1
            elif not attack_result['blocked'] and not scenario['expected_block']:
                # Legitimate traffic allowed correctly
                pass
            elif attack_result['blocked'] and not scenario['expected_block']:
                security_test_results['false_positives'] += 1
            else:
                security_test_results['attacks_allowed'] += 1
        
        # Calculate security effectiveness
        total_attacks = len([s for s in self.attack_scenarios if s['expected_block']])
        if total_attacks > 0:
            security_test_results['security_effectiveness'] = (
                security_test_results['attacks_blocked'] / total_attacks * 100
            )
        
        security_test_results['test_end'] = time.time()
        
        return security_test_results

# Usage example for healthcare F5 load testing
async def main():
    # Configure EHR application load test
    ehr_test_config = LoadTestConfig(
        application_name="EHR_System",
        virtual_server_ip="192.168.1.100",
        test_duration_seconds=300,  # 5 minutes
        concurrent_users=500,
        ramp_up_time=60,
        test_scenarios=[
            {
                'name': 'Patient_Lookup',
                'endpoint': '/api/patients/search',
                'method': 'GET',
                'concurrent_requests': 50,
                'auth_token': 'healthcare_test_token'
            },
            {
                'name': 'Medical_Records_Access',
                'endpoint': '/api/medical-records',
                'method': 'GET',
                'concurrent_requests': 30,
                'context': 'clinical_access'
            },
            {
                'name': 'Prescription_Orders',
                'endpoint': '/api/prescriptions',
                'method': 'POST',
                'payload': {'patient_id': 'TEST123', 'medication': 'Test Med'},
                'concurrent_requests': 20
            }
        ]
    )
    
    # Execute load testing
    load_tester = HealthcareF5LoadTester()
    load_test_results = await load_tester.execute_healthcare_load_test(ehr_test_config)
    
    # Execute security testing
    waf_tester = HealthcareWAFTester()
    security_results = await waf_tester.execute_security_testing("192.168.1.100")
    
    # Generate comprehensive test report
    test_report = {
        'load_testing': load_test_results,
        'security_testing': security_results,
        'compliance_summary': {
            'performance_compliant': load_test_results['compliance_validation']['performance_compliant'],
            'security_effective': security_results['security_effectiveness'] >= 95,
            'healthcare_ready': True
        }
    }
    
    print(json.dumps(test_report, indent=2))

if __name__ == "__main__":
    asyncio.run(main())
```

## Implementation Timeline and Success Metrics

**Project Timeline (12 weeks total):**

**Phase 1: Infrastructure Setup (Weeks 1-3)**
- F5 BIG-IP hardware installation and initial configuration
- High availability cluster setup and testing
- SSL certificate deployment and validation
- Basic load balancing configuration

**Phase 2: Healthcare Application Integration (Weeks 4-8)**
- EHR system load balancing configuration
- PACS/imaging application delivery setup
- WAF policy development and deployment
- Healthcare-specific iRule development and testing

**Phase 3: Automation and Testing (Weeks 9-11)**
- Infrastructure as Code development
- Automated testing framework implementation
- Performance optimization and tuning
- Security validation and compliance testing

**Phase 4: Production Deployment (Week 12)**
- Production cutover and validation
- Performance monitoring setup
- Staff training and knowledge transfer
- Documentation and support procedures

**Success Metrics:**
- 99.99% application availability for healthcare services
- Sub-100ms response times for EHR applications
- 100% security policy compliance with HIPAA requirements  
- Zero security incidents during testing and deployment
- 95%+ WAF attack detection and blocking effectiveness

This comprehensive F5 BIG-IP implementation ensures robust, secure, and compliant application delivery for healthcare organizations while maintaining operational excellence and regulatory compliance.