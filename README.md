demo-bundle/
├── README.md                                    (15 KB — the full walkthrough)
├── templates/
│   ├── 01-hello-world-site.yaml                 Lambda + API Gateway
│   ├── 02-hello-world-site-commented.yaml       same, with every line annotated
│   ├── 03-orchestrator-stack.yaml               two-Lambda S3 pipeline + demo VPC
│   └── 04-hello-world-cognito.yaml              full Cognito auth (working version)
├── deployments/
│   └── deployment-main.yaml                     Git sync deployment config
└── docs/
    └── git-sync-guide.md                        Git sync setup, step-by-step
