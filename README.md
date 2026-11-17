# Lab7-TF-Ans-Obs-Docker
Repo contains the next stages:
- Create an AWS architecture via Terraform using the next modules:
    - compute: used to create ec2 isntances
    - networking: used to create networking resources
    - security_groups: used to create security groups resources

- Configuration Management via Ansible using the next roles:
    - app_server: used on the private ec2 starting a http service via pyhton
    - docker_app: used on the public ec2 to create and start an nginx container and cAdvisor container for container metrics
    - grafana: used to install and start grafana on public ec2 to create dashboards
    - node_exporter: used on both nodes to fetch metrics
    - prometheus: used to install and start the service on the public ec2
    - swap: used on the public ec2 to be used as extra memory
    - web_proxy: used on the public ec2 as reverse proxy to the private ec2

## 🏛️ Architecture Overview

The infrastructure consists of a public-facing proxy layer and an isolated private application layer running on **Amazon Linux 2023**:

```
Internet
   │
   ▼
┌───────────────────┐
│ Internet Gateway  │
└─────────┬─────────┘
          │
┌─────────┴────────────────────────────────────────────────────────────────────────────┐
│ VPC (10.0.0.0/16)                                                                    │
│                                                                                      │
│   ┌──────────────────────────────────────────────────────────────────────────────┐   │
│   │ Public Subnet (10.0.1.0/24)                                                  │   │
│   │   - Route Table -> Internet Gateway                                          │   │
│   │                                                                              │   │
│   │   ┌──────────────────────────────────────────────────────────────────────┐   │   │
│   │   │ Public EC2: Web Proxy & Central Observability Node                   │   │   │
│   │   │  - Public IP                                                         │   │   │
│   │   │  - Nginx Reverse Proxy (Ports 80/443 for external traffic)           │   │   │
│   │   │  - Prometheus Server (Scraper) & Grafana Dashboards                  │   │   │
│   │   │  - Node Exporter (:9100), Swap, docker_app                           │   │   │
│   │   │  - Public Security Group                                             │   │   │
│   │   └──────────────────────────────────┬───────────────────────────────────┘   │   │
│   └──────────────────────────────────────┼───────────────────────────────────────┘   │
│                                          │                                           │
│             Internal Traffic             │ - App Traffic (Port 8080)                 │
│                                          │ - Metrics Scraping (:9100)                │
│                                          ▼                                           │
│   ┌──────────────────────────────────────────────────────────────────────────────┐   │
│   │ Private Subnet (10.0.2.0/24)                                                 │   │
│   │   - Route Table -> Local Only (No Direct Internet)                           │   │
│   │                                                                              │   │
│   │   ┌──────────────────────────────────────────────────────────────────────┐   │   │
│   │   │ Private EC2: Private Application Server                              │   │   │
│   │   │  - Private IP (10.0.2.x)                                             │   │   │
│   │   │  - Application Service (Listening on Port 8080)                      │   │   │
│   │   │  - Node Exporter (:9100)                                             │   │   │
│   │   │  - Private Security Group: Allow Port 8080 & 9100 ONLY from Public SG│   │   │
│   │   └──────────────────────────────────────────────────────────────────────┘   │   │
│   └──────────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────────┘
```
# Repository Layout
```
.
├── ansible
│   ├── ansible.cfg                             # Main configuration to be used with AWS
│   ├── group_vars
│   │   └── role_app_server.yml                 # Variables used for group role_app_server
│   ├── inventory
│   │   └── aws_ec2.yml                         # Dynamuc AWS EC2 inventory
│   ├── roles
│   │   ├── app_server
│   │   │   ├── tasks
│   │   │   │   └── main.yml                  # tasks to be perform for app_server role
│   │   │   └── templates
│   │   │       └── index.html.j2             # index file used for the app_server
│   │   ├── docker_app
│   │   │   ├── tasks
│   │   │   │   └── main.yml                  # tasks to be perform for docker_app role
│   │   │   └── templates
│   │   │       └── index.html.j2             # index file used for the app_server
│   │   ├── grafana
│   │   │   ├── handlers
│   │   │   ├── tasks
│   │   │   │   └── main.yml                  # tasks to be perform for grafana role
│   │   │   └── templates
│   │   ├── node_exporter
│   │   │   ├── handlers
│   │   │   │   └── main.yml                  # handlers to be perform for node_exporter role
│   │   │   └── tasks
│   │   │       └── main.yml                  # tasks to be perform for node_exporter role
│   │   ├── prometheus
│   │   │   ├── files
│   │   │   │   └── alert_rules.yml           # file that contains the alert rules for prometheus
│   │   │   ├── handlers
│   │   │   │   └── main.yml                  # handlers to be perform for prometheus role
│   │   │   ├── tasks
│   │   │   │   └── main.yml                  # tasks to be perform for prometheus role
│   │   │   └── templates
│   │   │       └── prometheus.yml.j2         # template used for prometheus role
│   │   ├── swap
│   │   │   └── tasks
│   │   │       └── main.yml                  # task to be perform for swap role
│   │   └── web_proxy
│   │       ├── handlers
│   │       │   └── main.yml                  # handlers to be perform for web_proxy role
│   │       ├── tasks
│   │       │   └── main.yml                  # tasks to perform for web_proxy role
│   │       └── templates
│   │           └── nginx_proxy.conf.j2       # template used for web_proxy role
│   └── site.yml                              # main playbook
├── README.md                                 # This README file
└── terraform
    ├── backend.tf                            # S3 backend used to store tf state file
    ├── main.tf                               # Main terraform file
    ├── modules
    │   ├── compute
    │   │   ├── main.tf                       # Main compute tf file
    │   │   ├── outputs.tf                    # Output compute tf file
    │   │   ├── variables.tf                  # Compute variables
    │   │   └── versions.tf                   # Compute tf required version
    │   ├── networking
    │   │   ├── main.tf                       # Main networking tf file
    │   │   ├── outputs.tf                    # Output networking tf file
    │   │   └── versions.tf                   # Networking tf required version
    │   └── security_groups
    │       ├── main.tf                       # Main security group tf file
    │       ├── outputs.tf                    # Output security group tf file
    │       ├── variables.tf                  # Security group variables
    │       └── versions.tf                   # Security group tf required version
    ├── providers.tf                          # AWS provider config
```
