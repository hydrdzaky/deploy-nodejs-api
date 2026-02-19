Markdown Live Preview
Reset
Copy
Export PDF

138139140141142143144145146147148149150151152153154155156157158159160161162163164165166167168169170171172173174175176177178179180181
│  │  - Data persistence                                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  Managed by Ansible:                                        │
│  - setup, nginx, node, docker, psql roles                   │
└─────────────────────────────────────────────────────────────┘
```

### Traffic Flow


Tools & Technologies
Infrastructure & DevOps
AWS EC2 - Compute instance for application hosting
Ansible - Infrastructure as Code for automated setup
Docker - Containerization for consistent deployments
GitHub Actions - CI/CD pipeline automation
Web Server & Proxy
Nginx - Reverse proxy and SSL termination
OpenSSL - SSL/TLS certificate management
Application
Node.js 25 - JavaScript runtime (Alpine 3.22 base)
npm - Package manager
Database
PostgreSQL 14+ - Relational database
Version Control & CI/CD
Git/GitHub - Version control
GitHub Actions - Workflow automation
appleboy/ssh-action - Remote SSH execution
Architecture
System Diagram
┌─────────────────────────────────────────────────────────────┐
│                      GitHub Repository                      │
│                    (main/master branch)                     │
└────────────────────────┬────────────────────────────────────┘
                         │ git push
                         ▼
┌────────────────────────────────────────────────────────────┐
│                   GitHub Actions CI/CD                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Checkout    │→ │  Build Image │→ │ Deploy Container │  │
│  │   Code       │  │  (Docker)    │  │  & Verify Logs   │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└────────────────────────┬───────────────────────────────────┘
                         │ SSH + Secrets
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   AWS EC2 Instance                          │
│                   (Ubuntu 22.04 LTS)                        │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   Nginx (Port 443)                     │ │
│  │         Reverse Proxy + SSL Termination                │ │
│  │     candidate-goul.qiscus.me → /api/cities             │ │
│  └────────────────────────────────────────────────────────┘ │
│           │                                                 │
│           ▼                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │          Docker Container (qiscus-api)                 │ │
│  │          Node.js App (Port 3000)                       │ │
│  │                                                        │ │
│  │  - Express.js API                                      │ │
│  │  - /api/cities endpoint handler                        │ │
│  │  - Request processing & logging                        │ │
│  └────────────────────────────────────────────────────────┘ │
│           │                                                 │
│           ▼                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         PostgreSQL (Port 5432)                         │ │
│  │                                                        │ │
│  │  - Database: qiscus_db                                 │ │
│  │  - User: qiscus_user                                   │ │
│  │  - Data persistence                                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  Managed by Ansible:                                        │
│  - setup, nginx, node, docker, psql roles                   │
└─────────────────────────────────────────────────────────────┘
Traffic Flow
User Request
    ↓
HTTPS (Port 443)
    ↓
Nginx Reverse Proxy
    ├─ SSL Termination
    └─ Route: /api/cities → localhost:3000
    ↓
Docker Container (localhost:3000)
    ├─ Node.js Application
    └─ Handle /api/cities request
    ↓
PostgreSQL Database
    ├─ Query execution
    └─ Return data
    ↓
Response ← Application ← Nginx ← User
🔄 Deployment Flow
Code Push (main/master branch)
        ↓
GitHub Actions Triggered
        ↓
Checkout Code
        ↓
SSH to EC2
        ↓
Build Docker Image
        ↓
Stop Old Container
        ↓
Run New Container
        ↓
View Logs
        ↓
✅ Deployment Complete
Quick Start
1️⃣ Infrastructure Setup (Ansible)
cd ansible-setup-server

# Update inventory with your EC2 IP
nano inventory.ini

# Run playbook
ansible-playbook -i inventory.ini playbook.yml -v
Installs: ✅ Node.js 25
✅ Docker Engine
✅ PostgreSQL 14+
✅ Nginx with SSL

2️⃣ Configure GitHub Secrets
gh secret set EC2_HOST --body "YOUR_EC2_PUBLIC_IP"
gh secret set EC2_USERNAME --body "ubuntu"
gh secret set EC2_PRIVATE_KEY < ~/.ssh/your_key.pem
gh secret set EC2_PORT --body "22"
gh secret set DB_USER --body "qiscus_user"
gh secret set DB_HOST --body "localhost"
gh secret set DB_NAME --body "qiscus_db"
gh secret set DB_PASSWORD --body "your_secure_password"
gh secret set DB_PORT --body "5432"
3️⃣ Deploy Application
# Just push your code
git push origin main

# GitHub Actions automatically:
# 1. Builds Docker image
# 2. Deploys to EC2
# 3. Shows logs
4️⃣ Access Application
https://candidate-goul.qiscus.me/api/cities
Directory Structure
qiscus-api/
├── ansible-setup-server/          # Infrastructure setup
│   ├── inventory.ini              # EC2 hosts
│   ├── playbook.yml               # Main playbook
│   └── roles/                     # Setup roles
│       ├── setup/
│       ├── nginx/
│       ├── node/
│       ├── docker/
│       └── psql/
├── .github/workflows/
│   └── pipeline.yml               # GitHub Actions CI/CD
├── Dockerfile                     # Container definition
├── package.json
├── app.js                         # Main application
├── init_db.sql                    # Database initialization
└── README.md                      # This file
Directory Structure
qiscus-api/
├── ansible-setup-server/          # Infrastructure setup
│   ├── inventory.ini              # EC2 hosts
│   ├── playbook.yml               # Main playbook
│   └── roles/                     # Setup roles
│       ├── setup/
│       ├── nginx/
│       ├── node/
│       ├── docker/
│       └── psql/
├── .github/workflows/
│   └── pipeline.yml               # GitHub Actions CI/CD
├── Dockerfile                     # Container definition
├── package.json
├── app.js                         # Main application
├── init_db.sql                    # Database initialization
└── README.md                      # This file
