# LOT 29: Network Architecture Services - Technical Design Guide

## 1. Architecture Design Methodology and Tools

### Enterprise Architecture Framework
**TOGAF 9.2 Implementation Approach:**
- Architecture Development Method (ADM) phases
- Business Architecture alignment with healthcare operations
- Information Systems Architecture for clinical workflows
- Technology Architecture supporting regulatory compliance

### Design Methodology Stack
- **Cisco SAFE Architecture:** Security reference framework
- **Zero Trust Network Architecture:** NIST SP 800-207 compliance
- **Healthcare Interoperability Standards:** HL7 FHIR, DICOM integration
- **Business Continuity Planning:** ISO 22301 framework

### Professional Design Tools

**Primary Architecture Tools:**
- **Enterprise Architect:** UML modeling and business process mapping
- **Lucidchart Enterprise:** Network topology and data flow documentation
- **Cisco Network Planner:** Capacity planning and performance modeling
- **NetBrain:** Dynamic network documentation and simulation

**Infrastructure as Code Tools:**
- **Terraform Enterprise:** Multi-cloud infrastructure provisioning
- **Ansible Tower:** Configuration management and orchestration
- **HashiCorp Vault:** Secrets management and certificate lifecycle
- **GitLab Enterprise:** Version control and CI/CD pipeline management

---

## 2. Multi-vendor Healthcare Network Architecture

### Greenfield Architecture Design

```
Healthcare Enterprise Network Architecture

Internet Gateway
       |
   [Palo Alto PA-5260]
   Perimeter Firewalls
       |
   DMZ Segment
       |
   [F5 BIG-IP i4800]
   Application Delivery
       |
Core Data Center
       |
   [Cisco ACI Fabric]
   ├── Patient Care Zone
   ├── Administrative Zone  
   ├── Medical Device Zone
   └── Research Zone
       |
Branch Locations
   [Cisco SD-WAN]
   ├── Clinic Sites (15x)
   ├── Imaging Centers (5x)
   └── Administrative Offices (8x)
```

### Detailed Network Topology

**Core Data Center Design:**
```
                    Internet (Redundant ISPs)
                          |
                   [PA-5260-HA-PAIR]
                    Primary | Secondary
                          |
                    [Core Switches]
                   Nexus-9K | Nexus-9K
                          |
              +----------+----------+
              |                     |
        [F5-BIG-IP-HA]        [ACI-Fabric]
        Primary|Secondary      Spine|Spine
              |                     |
        [DMZ-Services]         [Leaf-Layer]
        - Web Apps             - Patient Care
        - API Gateway          - Administration  
        - Partner Access       - Medical Devices
                              - Research Lab

Regional Sites (SD-WAN):
    Site-A: [ISR-4451] ── [Catalyst-9300] ── [Access-Points]
    Site-B: [ISR-4331] ── [Catalyst-9200] ── [Access-Points]
    Site-C: [ISR-4221] ── [Catalyst-9200] ── [Access-Points]
```

### Component Selection and Sizing

**Cisco ACI Fabric Specifications:**
```yaml
ACI_Fabric_Design:
  APIC_Cluster:
    - Model: APIC-SERVER-M3-LARGE
    - Quantity: 3 (High Availability)
    - Memory: 256GB RAM per node
    - Storage: 2.4TB SSD per node
    
  Spine_Switches:
    - Model: N9K-C9364C-GX
    - Quantity: 2 (Redundant pair)
    - Ports: 64x100G QSFP28
    - Throughput: 12.8Tbps per switch
    
  Leaf_Switches:
    - Model: N9K-C93180YC-EX
    - Quantity: 6 (3 per datacenter)
    - Ports: 48x25G + 6x100G uplinks
    - Features: ACI ready, MACSEC support
```

**Palo Alto Security Stack:**
```yaml
Perimeter_Security:
  Firewall_Cluster:
    - Model: PA-5260
    - Quantity: 2 (Active/Passive HA)
    - Throughput: 100Gbps
    - Sessions: 134M concurrent
    - Licensing: Threat Prevention, URL Filtering, WildFire
    
  Panorama_Management:
    - Model: M-600 Appliance
    - Quantity: 2 (HA pair)
    - Management: Centralized policy management
    - Logging: 20TB log storage capacity

  Prisma_Access:
    - Service: GlobalProtect Cloud Service  
    - Capacity: 5,000 concurrent users
    - Locations: 3 service connection points
    - Features: Zero Trust Network Access
```

**F5 Application Delivery:**
```yaml
Application_Delivery:
  BIG-IP_Cluster:
    - Model: BIG-IP i4800
    - Quantity: 2 (Active/Standby)
    - Throughput: 40Gbps L4, 20Gbps L7
    - SSL TPS: 100,000 bulk, 20,000 ECDSA
    - Modules: LTM, ASM, APM, AVR
    
  Services:
    - Load Balancing: Layer 4-7 with health monitoring
    - WAF Protection: OWASP Top 10, healthcare-specific rules
    - SSL Offloading: TLS 1.3 support with PFS
    - API Gateway: RESTful API protection and throttling
```

### Healthcare-Specific Network Zones

**1. Patient Care Zone Architecture:**
```
Patient Care Network Segment
├── Electronic Health Records (EHR)
│   ├── Epic Systems Cluster
│   ├── Database Servers (Always Encrypted)
│   └── Application Servers (Multi-tier)
├── Clinical Workstations
│   ├── Physician Workstations (VLAN 110)
│   ├── Nursing Stations (VLAN 111) 
│   └── Mobile Clinical Carts (VLAN 112)
└── Patient Monitoring Systems
    ├── Telemetry Monitors
    ├── Vital Signs Equipment
    └── Infusion Pump Network
```

**2. Medical Device Zone Configuration:**
```yaml
Medical_Device_Zone:
  Imaging_Network:
    - PACS Servers: 10.20.10.0/24
    - CT/MRI Scanners: 10.20.11.0/24  
    - X-ray Systems: 10.20.12.0/24
    - Ultrasound Equipment: 10.20.13.0/24
    
  Laboratory_Systems:
    - LIS Servers: 10.20.20.0/24
    - Analyzers: 10.20.21.0/24
    - Pathology Systems: 10.20.22.0/24
    
  Operating_Room_Network:
    - Surgical Equipment: 10.20.30.0/24
    - Anesthesia Machines: 10.20.31.0/24
    - Surgical Displays: 10.20.32.0/24
```

---

## 3. Migration Strategy and Risk Analysis

### Current State Assessment Framework

**Discovery and Documentation Phase:**
```python
#!/usr/bin/env python3
"""
Network Discovery and Documentation Script
Automated current state assessment tool
"""
import netmiko
import json
import ipaddress
from concurrent.futures import ThreadPoolExecutor

class NetworkDiscovery:
    def __init__(self, device_list):
        self.devices = device_list
        self.inventory = {}
        
    def discover_device_config(self, device):
        """Discover device configuration and capabilities"""
        try:
            connection = netmiko.ConnectHandler(**device)
            
            # Gather device information
            device_info = {
                'hostname': connection.send_command('show hostname'),
                'version': connection.send_command('show version'),
                'interfaces': connection.send_command('show interface status'),
                'vlans': connection.send_command('show vlan brief'),
                'routing': connection.send_command('show ip route summary'),
                'acls': connection.send_command('show access-lists')
            }
            
            connection.disconnect()
            return device_info
            
        except Exception as e:
            return {'error': str(e), 'device': device['ip']}
    
    def analyze_traffic_patterns(self, interface_data):
        """Analyze traffic patterns for migration planning"""
        traffic_analysis = {}
        
        for interface in interface_data:
            if interface['utilization'] > 70:
                traffic_analysis[interface['name']] = {
                    'status': 'high_utilization',
                    'recommendation': 'Priority migration candidate',
                    'bandwidth_requirement': interface['speed'] * 1.5
                }
        
        return traffic_analysis

# Usage for current state assessment
device_inventory = [
    {'device_type': 'cisco_ios', 'ip': '192.168.1.1', 'username': 'admin', 'password': 'secure'},
    {'device_type': 'cisco_nxos', 'ip': '192.168.1.2', 'username': 'admin', 'password': 'secure'}
]

discovery = NetworkDiscovery(device_inventory)
current_state = discovery.discover_all_devices()
```

### Migration Planning Framework

**Phase 1: Infrastructure Preparation (Weeks 1-4)**
```yaml
Infrastructure_Prep:
  Data_Center_Setup:
    - ACI fabric installation and initial configuration
    - APIC cluster deployment and licensing
    - Spine-leaf cabling and port assignments
    - Management network configuration
    
  Security_Platform_Setup:
    - Palo Alto firewall installation and HA configuration
    - Panorama deployment and initial policies
    - F5 BIG-IP cluster setup and module licensing
    - SSL certificate deployment and management
    
  Testing_Environment:
    - Parallel lab environment setup
    - Configuration validation procedures
    - Performance baseline establishment
    - Security policy testing
```

**Phase 2: Application Migration (Weeks 5-12)**
```yaml
Application_Migration_Sequence:
  Critical_Applications:
    Week_5-6:
      - EHR system migration (Epic)
      - Patient monitoring systems
      - Laboratory information systems
    
  Administrative_Systems:
    Week_7-8:
      - Financial and billing systems
      - Human resources applications
      - Email and collaboration tools
    
  Medical_Devices:
    Week_9-10:
      - PACS and imaging systems
      - Pharmacy management systems
      - Medical IoT devices
    
  External_Services:
    Week_11-12:
      - Partner connectivity
      - Remote access services
      - Internet-facing applications
```

### Comprehensive Risk Analysis

**Technical Risk Assessment:**
```yaml
Risk_Matrix:
  High_Risk_Items:
    - Network_Downtime:
        Impact: "Critical patient care disruption"
        Probability: "Medium"
        Mitigation: "Parallel deployment with rollback procedures"
        
    - Data_Loss:
        Impact: "HIPAA violation and patient safety"
        Probability: "Low"
        Mitigation: "Comprehensive backup and replication strategy"
        
    - Security_Breach:
        Impact: "Regulatory fines and reputation damage"
        Probability: "Medium"
        Mitigation: "Zero-trust architecture and continuous monitoring"
  
  Medium_Risk_Items:
    - Performance_Degradation:
        Impact: "Reduced clinical efficiency"
        Probability: "Medium"  
        Mitigation: "Performance testing and capacity planning"
        
    - Integration_Issues:
        Impact: "Application functionality loss"
        Probability: "High"
        Mitigation: "Extensive pre-migration testing"

  Low_Risk_Items:
    - Staff_Training:
        Impact: "Temporary productivity loss"
        Probability: "High"
        Mitigation: "Comprehensive training program"
```

**Business Continuity Planning:**
```python
#!/usr/bin/env python3
"""
Business Continuity Validation Script
Ensures critical services remain operational during migration
"""
class BusinessContinuityValidator:
    def __init__(self):
        self.critical_services = [
            {'name': 'EHR_System', 'rto': 30, 'rpo': 5},
            {'name': 'Patient_Monitoring', 'rto': 10, 'rpo': 1},
            {'name': 'PACS_Imaging', 'rto': 60, 'rpo': 15},
            {'name': 'Laboratory_Systems', 'rto': 45, 'rpo': 10}
        ]
    
    def validate_service_availability(self, service_name):
        """Validate service meets RTO/RPO requirements"""
        # Implementation for service health checks
        # Database connectivity validation
        # Application response time measurement
        # Failover procedure testing
        pass
    
    def execute_rollback_procedure(self, failed_service):
        """Execute automated rollback for failed migration"""
        rollback_steps = [
            "Stop new service configuration",
            "Restore previous network configuration", 
            "Validate service restoration",
            "Update change management system"
        ]
        
        for step in rollback_steps:
            self.execute_rollback_step(step, failed_service)
```

---

## 4. Infrastructure Automation Framework

### Terraform Infrastructure Provisioning

**Main Infrastructure Configuration:**
```hcl
# main.tf - Healthcare Network Infrastructure
terraform {
  required_version = ">= 1.0"
  required_providers {
    aci = {
      source  = "CiscoDevNet/aci"
      version = "~> 2.0"
    }
    panos = {
      source  = "PaloAltoNetworks/panos"
      version = "~> 1.11"
    }
    bigip = {
      source  = "F5Networks/bigip"
      version = "~> 1.16"
    }
  }
}

# ACI Provider Configuration
provider "aci" {
  username = var.aci_username
  password = var.aci_password
  url      = var.aci_url
  insecure = true
}

# Healthcare Tenant Configuration
resource "aci_tenant" "healthcare_prod" {
  name        = "HEALTHCARE_PROD"
  description = "Production tenant for healthcare applications"
}

# VRF Configuration for Patient Care Zone
resource "aci_vrf" "patient_care_vrf" {
  tenant_dn = aci_tenant.healthcare_prod.id
  name      = "PATIENT_CARE_VRF"
  
  relation_fv_rs_ctx_to_bgp_ctx_af_pol = [
    aci_bgp_address_family_context.ipv4_ucast.id
  ]
}

# Bridge Domain for EHR Systems
resource "aci_bridge_domain" "ehr_systems_bd" {
  tenant_dn   = aci_tenant.healthcare_prod.id
  name        = "BD_EHR_SYSTEMS"
  relation_fv_rs_ctx = aci_vrf.patient_care_vrf.id
  
  # Enable unicast routing
  unicast_route = "yes"
  
  # ARP flooding disabled for security
  arp_flood = "no"
  
  # Unknown multicast flooding disabled
  multi_dst_pkt_act = "bd-flood"
}

# Application Profile for Patient Care Applications
resource "aci_application_profile" "patient_care_ap" {
  tenant_dn = aci_tenant.healthcare_prod.id
  name      = "AP_PATIENT_CARE"
}

# EPG for EHR Servers
resource "aci_application_epg" "ehr_servers_epg" {
  application_profile_dn = aci_application_profile.patient_care_ap.id
  name                  = "EPG_EHR_SERVERS"
  relation_fv_rs_bd     = aci_bridge_domain.ehr_systems_bd.id
  
  # Preferred group membership for optimized forwarding
  pref_gr_memb = "include"
}
```

**Palo Alto Configuration Module:**
```hcl
# modules/palo-alto/main.tf
resource "panos_ethernet_interface" "inside_interface" {
  name           = "ethernet1/1"
  vsys           = "vsys1"
  mode           = "layer3"
  enable_dhcp    = false
  static_ips     = ["192.168.100.1/24"]
  management_profile = panos_interface_management_profile.ping_profile.name
}

# Security zones configuration
resource "panos_zone" "patient_care_zone" {
  name           = "patient-care-zone"
  mode           = "layer3"
  interfaces     = [panos_ethernet_interface.inside_interface.name]
  enable_user_id = true
  
  # Healthcare-specific zone protection
  zone_profile = panos_zone_protection_profile.healthcare_protection.name
}

# Healthcare security policies
resource "panos_security_policy" "healthcare_policies" {
  rule {
    name                  = "Allow-EHR-Database-Access"
    source_zones          = ["patient-care-zone"]
    source_addresses      = ["EPG-PCZ-Workstations"]
    destination_zones     = ["data-center-zone"]
    destination_addresses = ["EPG-EHR-Servers"]
    applications          = ["ssl", "ms-sql-db"]
    services              = ["application-default"]
    categories            = ["any"]
    action                = "allow"
    
    # Enable logging for HIPAA compliance
    log_setting = "healthcare-audit-log"
  }
  
  rule {
    name                  = "Block-Internet-Medical-Devices"
    source_zones          = ["medical-device-zone"]
    source_addresses      = ["any"]
    destination_zones     = ["internet-zone"]
    destination_addresses = ["any"]
    applications          = ["any"]
    services              = ["any"]
    categories            = ["any"]
    action                = "deny"
    
    log_setting = "security-log-profile"
  }
}
```

**F5 BIG-IP Automation:**
```hcl
# modules/f5-bigip/main.tf
resource "bigip_ltm_pool" "ehr_application_pool" {
  name               = "/Common/pool_ehr_applications"
  load_balancing_mode = "round-robin"
  
  # Health monitoring for EHR applications
  monitors = ["/Common/https_443"]
  
  # Pool members for EHR cluster
  members {
    address = "10.10.10.100"
    port    = 443
  }
  
  members {
    address = "10.10.10.101"  
    port    = 443
  }
}

# Virtual server for EHR applications
resource "bigip_ltm_virtual_server" "ehr_vs" {
  pool                       = bigip_ltm_pool.ehr_application_pool.name
  name                       = "/Common/vs_ehr_applications"
  destination               = "192.168.1.100"
  port                      = 443
  protocols                 = ["tcp"]
  
  # SSL profile for healthcare compliance
  profiles = [
    "/Common/tcp-lan-optimized",
    "/Common/healthcare-ssl-profile"
  ]
  
  # iRule for healthcare-specific processing
  irules = [bigip_ltm_irule.healthcare_security_irule.name]
}

# Healthcare security iRule
resource "bigip_ltm_irule" "healthcare_security_irule" {
  name = "/Common/healthcare_security_processing"
  
  irule = <<EOF
when HTTP_REQUEST {
    # Log all healthcare application access for audit
    log local0. "Healthcare access: [IP::client_addr] -> [HTTP::host][HTTP::uri]"
    
    # Block non-TLS healthcare traffic
    if { [TCP::local_port] != 443 } {
        reject
    }
    
    # Validate healthcare-specific headers
    if { [HTTP::header exists "X-Healthcare-Context"] } {
        # Allow authenticated healthcare traffic
        pool pool_ehr_applications
    } else {
        # Redirect to authentication portal
        HTTP::respond 302 Location "https://auth.healthcare.local/login"
    }
}
EOF
}
```

### Ansible Configuration Management

**Main Playbook Structure:**
```yaml
# site.yml - Healthcare Network Deployment
---
- name: Deploy Healthcare Network Infrastructure
  hosts: localhost
  connection: local
  gather_facts: no
  
  vars:
    # Healthcare-specific variables
    healthcare_tenant: "HEALTHCARE_PROD"
    compliance_logging: true
    encryption_required: true
  
  roles:
    - role: aci-tenant-deployment
      vars:
        tenant_name: "{{ healthcare_tenant }}"
        
    - role: security-policy-deployment
      vars:
        compliance_mode: "HIPAA"
        
    - role: monitoring-deployment
      vars:
        audit_logging: "{{ compliance_logging }}"

# roles/aci-tenant-deployment/tasks/main.yml
- name: Deploy Healthcare Tenant Configuration
  cisco.aci.aci_tenant:
    host: "{{ aci_host }}"
    username: "{{ aci_username }}"
    password: "{{ aci_password }}"
    tenant: "{{ tenant_name }}"
    description: "Healthcare production tenant with HIPAA compliance"
    state: present
  register: tenant_result

- name: Create Patient Care VRF
  cisco.aci.aci_vrf:
    host: "{{ aci_host }}"
    username: "{{ aci_username }}"
    password: "{{ aci_password }}"
    tenant: "{{ tenant_name }}"
    vrf: "PATIENT_CARE_VRF"
    policy_control_preference: "enforced"
    policy_control_direction: "ingress"
    state: present

- name: Deploy Healthcare Bridge Domains
  cisco.aci.aci_bd:
    host: "{{ aci_host }}"
    username: "{{ aci_username }}"
    password: "{{ aci_password }}"
    tenant: "{{ tenant_name }}"
    bd: "{{ item.name }}"
    vrf: "{{ item.vrf }}"
    enable_routing: "{{ item.routing }}"
    arp_flooding: "{{ item.arp_flood }}"
    l2_unknown_unicast: "{{ item.l2_unknown }}"
    state: present
  loop:
    - { name: "BD_EHR_SYSTEMS", vrf: "PATIENT_CARE_VRF", routing: "yes", arp_flood: "no", l2_unknown: "proxy" }
    - { name: "BD_MEDICAL_DEVICES", vrf: "PATIENT_CARE_VRF", routing: "yes", arp_flood: "no", l2_unknown: "flood" }
    - { name: "BD_ADMINISTRATION", vrf: "ADMIN_VRF", routing: "yes", arp_flood: "no", l2_unknown: "proxy" }
```

### GitOps Workflow Implementation

**GitLab CI/CD Pipeline:**
```yaml
# .gitlab-ci.yml
stages:
  - validate
  - test
  - deploy-staging
  - security-scan
  - deploy-production

variables:
  TERRAFORM_VERSION: "1.3.0"
  ANSIBLE_VERSION: "6.0.0"

validate_terraform:
  stage: validate
  image: hashicorp/terraform:$TERRAFORM_VERSION
  script:
    - terraform init
    - terraform validate
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan
    expire_in: 1 hour

validate_ansible:
  stage: validate  
  image: quay.io/ansible/ansible-runner:latest
  script:
    - ansible-playbook --syntax-check site.yml
    - ansible-lint playbooks/

test_lab_deployment:
  stage: test
  image: quay.io/ansible/ansible-runner:latest
  script:
    - ansible-playbook -i lab-inventory site.yml --check --diff
  environment:
    name: lab
  only:
    - merge_requests
    - develop

security_compliance_scan:
  stage: security-scan
  image: bridgecrew/checkov:latest
  script:
    - checkov -f main.tf --framework terraform --check CKV_* 
    - checkov -f ansible/playbooks --framework ansible
  allow_failure: false

deploy_to_production:
  stage: deploy-production
  image: hashicorp/terraform:$TERRAFORM_VERSION
  script:
    - terraform init
    - terraform apply tfplan
    - ansible-playbook -i production-inventory site.yml
  environment:
    name: production
  when: manual
  only:
    - main
```

## Implementation Timeline and Success Metrics

**Project Timeline (20 weeks total):**

**Phase 1: Analysis and Design (Weeks 1-4)**
- Current state assessment and documentation
- Future state architecture design
- Risk analysis and mitigation planning
- Stakeholder alignment and approval

**Phase 2: Infrastructure Build (Weeks 5-12)**  
- Data center infrastructure deployment
- Security platform configuration
- Network automation development
- Testing and validation procedures

**Phase 3: Migration Execution (Weeks 13-18)**
- Phased application migration
- Performance monitoring and optimization
- Security policy refinement
- Staff training and knowledge transfer

**Phase 4: Optimization and Handover (Weeks 19-20)**
- Performance tuning and capacity optimization
- Documentation finalization
- Support transition procedures
- Project closure and lessons learned

**Success Metrics:**
- Zero unplanned downtime during business hours
- 99.9% application availability post-migration
- Sub-10ms latency improvement for critical applications
- 100% security policy compliance validation
- Staff certification completion within 30 days

This comprehensive architecture approach ensures robust, secure, and compliant network infrastructure aligned with healthcare industry requirements while maintaining operational excellence and regulatory compliance.