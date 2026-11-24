# Network & Security Projects

Hands-on projects in network engineering, security and cloud infrastructure. Each folder holds a PDF report. The two Cisco projects also include every router and switch configuration as plain text (`show running-config`) and the Packet Tracer file.

| Project | Stack | Report |
| --- | --- | --- |
| [Secure network services lab](#secure-network-services-lab) | Apache, OpenSSL CA, ELK Stack, Squid, iptables | [PDF](secure-network-services-lab/Secure_Network_Services_Lab_Report.pdf) |
| [Campus occupancy dashboard](#campus-occupancy-dashboard) | AWS ECS Fargate, ALB, DynamoDB, Cognito, Terraform, GitHub Actions | [PDF](campus-occupancy-dashboard/Campus_Occupancy_Dashboard_Report.pdf) |
| [Enterprise network architecture](#enterprise-network-architecture) | Cisco: multi-area OSPF, eBGP, VLSM, DHCP, NAT, ACL | [PDF](enterprise-network-architecture/Enterprise_Network_Architecture_Report.pdf) |
| [Resto Network](#resto-network) | Cisco: VLANs, router-on-a-stick, ACL, SSH, WPA2 Wi-Fi | [PDF](resto-network/Resto_Network_Report.pdf) |

---

## Secure network services lab

On one Kali Linux VM, I served the local domain `rithish.local` over HTTPS with Apache, using a server certificate signed by my own OpenSSL certificate authority and verified with OpenSSL, curl and Firefox. Apache and Squid proxy logs go through Logstash into Elasticsearch 7.17, where Kibana searches and charts them. A DMZ is simulated with a Linux network namespace joined by a veth pair: iptables lets only the host reach the DMZ web server on port 8080 and drops traffic from a second "client" namespace. The host firewall sets INPUT and FORWARD to DROP, allows SSH only from the admin machine, and is saved with netfilter-persistent.

```mermaid
flowchart LR
    admin["Admin laptop<br/>192.168.64.1"] -->|"SSH, only allowed source"| fw
    subgraph vm["Kali Linux VM - 192.168.64.9"]
        fw{{"iptables<br/>INPUT and FORWARD: DROP"}}
        fw -->|"443"| apache["Apache<br/>https://rithish.local<br/>cert from local CA"]
        fw -->|"3128"| squid["Squid proxy"]
        apache -->|"access logs"| logstash["Logstash"]
        squid -->|"access logs"| logstash
        logstash --> es[("Elasticsearch")]
        es --> kibana["Kibana dashboards"]
        hostveth["Host veth<br/>10.10.10.1"] -->|"allowed"| web
        client["Client namespace<br/>10.10.10.3"] -.->|"dropped by iptables"| web
        subgraph dmz["DMZ namespace"]
            web["Web server :8080"]
        end
    end
```

**Files:** [Report (PDF)](secure-network-services-lab/Secure_Network_Services_Lab_Report.pdf)

---

## Campus occupancy dashboard

A team project (4 people): a Spring Boot API and dashboard that track room occupancy across a campus, deployed on AWS in Paris (eu-west-3). Two to three ECS Fargate tasks run behind an Application Load Balancer across two availability zones and scale on CPU. Users sign in with Cognito and the API checks their JWT on every request; data sits in DynamoDB and an EventBridge-triggered Lambda exports it to S3 every day. All infrastructure is Terraform with remote state in S3, and GitHub Actions builds the image, applies Terraform and redeploys ECS through OIDC, with no stored AWS keys. The compute layer started on EC2 with an Auto Scaling Group and was migrated to Fargate after a design review.

```mermaid
flowchart LR
    push["git push to main"] --> gha["GitHub Actions"]
    gha -->|"build and push image"| hub["Docker Hub<br/>private image"]
    gha -->|"OIDC role: terraform apply,<br/>ecs update-service"| ecs
    user["Browser"] -->|"sign in"| cognito
    user -->|"HTTP :80"| alb
    subgraph aws["AWS eu-west-3 - VPC across 2 AZs - defined in Terraform"]
        alb["Application Load Balancer"] -->|":8080"| ecs["ECS Fargate service<br/>2-3 Spring Boot tasks"]
        cognito["Cognito user pool"]
        ecs -.->|"validate JWT"| cognito
        ecs --> ddb[("DynamoDB<br/>events, rooms")]
        rule["EventBridge daily rule"] --> lambda["Lambda export"]
        ddb -.->|"daily export"| lambda
        lambda --> s3[("S3 backups")]
        ecs --> cw["CloudWatch logs and alarms"]
        cw --> sns["SNS email alerts"]
    end
    hub -->|"image pull"| ecs
```

**Files:** [Report (PDF)](campus-occupancy-dashboard/Campus_Occupancy_Dashboard_Report.pdf)

---

## Enterprise network architecture

A simulated company network (AS100) linked by eBGP to an outside network (AS200), built in Cisco Packet Tracer. Four routers run multi-area OSPF: R2 and R3 form the backbone (Area 0), and R0 and R1 are area border routers for the user LANs in Area 1. Three rooms share 172.16.1.0/24 through VLSM (/25, /26, /27) and get their addresses from DHCP pools on R0. R2 redistributes routes between OSPF and BGP, runs PAT toward AS200, and connects a server LAN with an HTTP/DNS server and an SMTP/POP3 mail server.

```mermaid
flowchart LR
    subgraph as100["AS100 - company"]
        subgraph area0["OSPF Area 0"]
            R3["R3"]
            R2["R2<br/>ASBR, NAT"]
            srv["Server LAN 192.168.5.0/24<br/>HTTP + DNS, Mail"]
        end
        subgraph area1["OSPF Area 1"]
            R0["R0<br/>ABR, DHCP"]
            R1["R1<br/>ABR"]
            rooms["Rooms A, B, C#amp;D<br/>172.16.1.0/24 as /25, /26, /27"]
            lan4["LAN 192.168.4.0/24"]
        end
    end
    subgraph as200["AS200"]
        RB["Router0"] --- lan200["LAN 172.16.0.0/16"]
    end
    R0 ---|"192.168.3.0/24"| R3
    R3 ---|"192.168.2.0/24"| R2
    R1 ---|"192.168.1.0/24"| R2
    R0 ---|"192.168.0.0/24"| R1
    R2 ---|"eBGP 222.223.50.0/24"| RB
    R2 --- srv
    R0 --- rooms
    R1 --- lan4
```

**Files:** [Report (PDF)](enterprise-network-architecture/Enterprise_Network_Architecture_Report.pdf) · [Router configs](enterprise-network-architecture/configs/) · [Packet Tracer file](enterprise-network-architecture/enterprise-network.pkt)

---

## Resto Network

A small restaurant network split into four VLANs: manager (10), counter/POS (20), customer Wi-Fi (30) and switch management (99). A Cisco 2911 router routes between them over an 802.1Q trunk (router-on-a-stick) and runs DHCP for the three user VLANs. An extended ACL on the guest sub-interface blocks customers from the internal VLANs while keeping their Internet access. The Catalyst 3560 switch accepts management only over SSH v2 with local user accounts, and a WPA2 access point serves the guest SSID on VLAN 30.

```mermaid
flowchart TB
    isp["Internet / ISP"] --- r1["R1 - Cisco 2911<br/>router-on-a-stick, DHCP, ACL"]
    r1 ===|"802.1Q trunk: VLANs 10, 20, 30, 99"| sw["MainSwitch - Catalyst 3560<br/>SSH v2 management on VLAN 99"]
    sw --- v10["VLAN 10 MANAGER<br/>192.168.10.0/24<br/>PC + printer"]
    sw --- v20["VLAN 20 COUNTER<br/>192.168.20.0/24<br/>POS PC + printer"]
    sw --- ap["Access point<br/>SSID Resto, WPA2"]
    ap --- v30["VLAN 30 CUSTOMER-WIFI<br/>192.168.30.0/24<br/>guest laptops"]
    v30 -. "ACL: no access to VLANs 10, 20, 99" .-> r1
```

**Files:** [Report (PDF)](resto-network/Resto_Network_Report.pdf) · [Router and switch configs](resto-network/configs/) · [Packet Tracer file](resto-network/resto-network.pkt)

---

Passwords and secrets in the exported configs, and the Wi-Fi passphrase in the Resto report, are replaced with placeholders.
