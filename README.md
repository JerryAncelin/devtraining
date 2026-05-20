# CloudFormation Demo Bundle

A progressive walkthrough of AWS serverless infrastructure using CloudFormation,
moving from a single Lambda + API Gateway to a full Cognito-authenticated
application. Each template builds on lessons from the previous one and can be
deployed independently.

## Contents

```
demo-bundle/
├── README.md                                    ← you are here
├── templates/
│   ├── 01-hello-world-site.yaml                 ← Lambda + API Gateway micro site
│   ├── 02-hello-world-site-commented.yaml       ← same as 01, heavily annotated
│   ├── 03-orchestrator-stack.yaml               ← two Lambdas + S3 + demo VPC
│   └── 04-hello-world-cognito.yaml              ← full Cognito auth demo
├── deployments/
│   └── deployment-main.yaml                     ← CloudFormation Git sync config
└── docs/
    └── git-sync-guide.md                        ← step-by-step Git sync setup
```

## Prerequisites

- AWS account with permission to create IAM roles, Lambda, API Gateway,
  Cognito, S3, EC2 (VPC), and CloudFormation stacks
- AWS CLI v2 installed and authenticated (`aws configure` or `aws sso login`)
- Region: examples target `us-east-1` — change with `--region` if needed
- Optional: a GitHub repo if you want to try the Git sync workflow

---

## Template 1 — Hello-world micro site

**File:** `templates/01-hello-world-site.yaml`
**Annotated version:** `templates/02-hello-world-site-commented.yaml`

The simplest possible serverless website: a Lambda function that returns a
styled HTML page, fronted by API Gateway. This is the foundation everything
else builds on.

### What it creates

- IAM execution role for the Lambda
- Lambda function with inline Node.js code (no S3 packaging needed)
- API Gateway REST API with `GET /` mapped to the Lambda
- Lambda invoke permission so API Gateway can call the function
- Deployment + stage so the API is publicly callable

### Deploy

```bash
aws cloudformation deploy \
  --template-file templates/01-hello-world-site.yaml \
  --stack-name hello-world-site \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

### Test

The stack output `WebsiteUrl` gives you the URL. Open it in a browser — you'll
see a styled "Hello, World!" page.

### Key concepts demonstrated

- **`AWS_PROXY` Lambda integration** — API Gateway forwards the entire request
  to Lambda and returns whatever Lambda returns (in the `{ statusCode, headers,
  body }` shape).
- **`IntegrationHttpMethod: POST` even for GET endpoints** — API Gateway
  always invokes Lambda over POST regardless of the client's HTTP method.
- **`AWS::Lambda::Permission`** — the resource-based policy that lets API
  Gateway invoke the Lambda. Forgetting this is the #1 cause of 500 errors.
- **`Deployment` `DependsOn` `Method`** — without it, CloudFormation may
  deploy an empty API and you get "Missing Authentication Token" errors.

The `02-hello-world-site-commented.yaml` version explains every CloudFormation
intrinsic function (`!Ref`, `!Sub`, `!GetAtt`), every API Gateway requirement,
and the wiring between them. Use it as a reference when reading the other
templates.

---

## Template 3 — Orchestrator + processor + demo VPC

**File:** `templates/03-orchestrator-stack.yaml`

Steps up to a more realistic pattern: two Lambdas where one dynamically creates
infrastructure and triggers the other via S3 events. Includes a demo VPC
purely to illustrate CloudFormation changesets (especially under Git sync).

### What it creates

- **Processor Lambda** — invoked by S3 event notifications, reads and "processes" uploaded JSON
- **Orchestrator Lambda** — when manually invoked:
  1. Creates a uniquely-named S3 bucket
  2. Grants S3 permission to invoke the processor Lambda
  3. Configures the bucket's event notification → processor Lambda
  4. Uploads synthetic JSON data (which triggers the processor)
- **IAM roles** for both Lambdas with minimum-needed scoping
- **Demo VPC** with two tags — exists only to illustrate changeset behavior

### Deploy

```bash
aws cloudformation deploy \
  --template-file templates/03-orchestrator-stack.yaml \
  --stack-name orchestrator-demo \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

### Test

```bash
# Trigger the end-to-end flow
aws lambda invoke \
  --function-name orchestrator-demo-orchestrator \
  --region us-east-1 \
  response.json

cat response.json
# Returns: { "statusCode": 200, "bucket": "synthetic-data-...", "key": "..." }
```

Then check the processor Lambda's logs:

```bash
aws logs tail /aws/lambda/orchestrator-demo-processor \
  --follow --region us-east-1
```

You should see the processor logging "Triggered by s3://..." within a second
of the orchestrator finishing.

### Demonstrating changesets with the VPC

This is the cleanest resource for showing CloudFormation's changeset behavior:

- **In-place update (no replacement)** — Edit a tag value on `DemoVpc` and
  redeploy. Changeset shows `Modify` with `Requires Replacement: False`.
- **Replacement update** — Change `VpcCidr` from `10.0.0.0/16` to `10.1.0.0/16`
  and redeploy. Changeset shows `Modify` with `Requires Replacement: True` —
  CloudFormation deletes the old VPC and creates a new one with a different ID.
- **No-op deploy** — Change a comment in the template. The changeset shows no
  resource changes; useful for proving Git sync reconciliation works even when
  nothing actually changes.

### Key concepts demonstrated

- **Lambda creating AWS resources at runtime** (vs. CloudFormation creating
  them at deploy time). Useful for self-service patterns but adds operational
  complexity — these dynamic buckets aren't tracked by CloudFormation.
- **`AWS::Lambda::Permission` added at runtime via `boto3`** — same idea as
  the static permission in template 1, but issued dynamically per bucket.
- **S3 → Lambda event integration** — `put_bucket_notification_configuration`
  must run *after* the Lambda permission exists, or S3 rejects the call.
- **Scoped IAM** — the orchestrator's role only grants S3 actions on buckets
  matching `${BucketPrefix}-*`, not all buckets in the account.

---

## Template 4 — Full Cognito authentication demo

**File:** `templates/04-hello-world-cognito.yaml`

The full end-to-end Cognito flow: User Pool + Hosted UI + Identity Pool +
protected API + temporary AWS credentials. Designed to make every interaction
point in the auth flow visible.

### What it creates

- **Cognito User Pool** — the user directory (where accounts live)
- **User Pool Domain** — provides the Hosted UI login page
- **User Pool Client** — the OAuth client config (callback URLs, allowed scopes)
- **Cognito Identity Pool** — issues temporary AWS credentials to authenticated users
- **Authenticated IAM Role** — what those temp credentials can do
- **Lambda function** — serves both the public HTML page and the protected `/whoami` endpoint
- **API Gateway** with two methods:
  - `GET /` — public, returns the HTML SPA
  - `GET /whoami` — protected by Cognito JWT authorizer
- **Cognito Authorizer** — validates JWTs against the User Pool

### Deploy

```bash
aws cloudformation deploy \
  --template-file templates/04-hello-world-cognito.yaml \
  --stack-name hello-cognito \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

> **Important:** Stack name and `AppName` parameter must NOT contain `aws`,
> `amazon`, or `cognito` — these are reserved words in Cognito domain prefixes
> and produce "Invalid request provided" errors. Use names like `helloauth`
> or `lambdademo`.

### Demo flow

1. Open the `WebsiteUrl` from the stack outputs.
2. Click **Sign in (Hosted UI)** — redirects to the Cognito login page.
3. Click **Sign up**, register with email + password. Cognito sends a
   verification code via email; enter it.
4. After verification, you're redirected back to the page with an OAuth
   `?code=`. The page exchanges it for ID + Access tokens.
5. The **ID Token claims** panel now shows your decoded JWT (email, sub, etc).
6. Click **Call protected `/whoami`** — sends the JWT to API Gateway, which
   validates it via the Cognito authorizer, then forwards to Lambda. Returns
   the verified claims. Try this *before* signing in too — you'll get a 401,
   proving the authorizer works.
7. Click **Get temporary AWS credentials** — exchanges the ID token via the
   Identity Pool for short-lived AWS access key/secret/session-token tied to
   the `AuthenticatedRole`. These could be used to call S3, DynamoDB, etc.
   directly from the browser.

### Interaction points mapped to the code

| # | Interaction | Where in the template |
|---|---|---|
| 1 | Page load | `RootMethod` → Lambda → returns HTML |
| 2 | Click "Sign in" | JS `login()` uses `HOSTED_UI_DOMAIN` + `CLIENT_ID` (Lambda env vars) |
| 3 | Hosted UI sign-up/sign-in | Provided by `UserPool` + `UserPoolDomain` |
| 4 | Redirect back with `?code=` | Cognito honors `CallbackURLs` in `UserPoolClient` |
| 5 | Code → tokens exchange | JS `handleRedirect()` POSTs to `/oauth2/token` |
| 6 | Show decoded ID token claims | JS `showClaims()` — base64-decodes the JWT |
| 7 | Call `/whoami` with JWT | JS `callWhoami()` → API Gateway → `CognitoAuthorizer` → Lambda |
| 8 | Get temp AWS credentials | JS `getAwsCreds()` → `cognito-identity:GetId` + `GetCredentialsForIdentity` → `IdentityPool` issues creds tied to `AuthenticatedRole` |

### Key concepts demonstrated

- **User Pool vs. Identity Pool** — they sound similar but do different things.
  User Pool answers "who is this person" (authentication). Identity Pool
  answers "what AWS perms do they get" (federated AWS authorization).
- **Hosted UI** — Cognito-provided login/signup pages, configured purely
  through the `UserPoolDomain` + `UserPoolClient` resources. No frontend
  auth code needed.
- **Cognito Authorizer on API Gateway** — validates JWTs server-side. By the
  time your Lambda runs, the token is guaranteed valid and the verified
  claims arrive in `event.requestContext.authorizer.claims`.
- **`AssumeRoleWithWebIdentity` trust policy** — the magic that lets a JWT
  from Cognito be exchanged for AWS STS credentials. Note the trust
  conditions on `aud` (identity pool ID) and `amr` (must be `authenticated`).
- **Avoiding circular dependencies** — the original version had
  `UserPoolClient DependsOn: ApiStage` which created a cycle through
  `WhoamiMethod → CognitoAuthorizer → UserPool → UserPoolClient`. The fix:
  reference the API ID directly with `!Sub`, since CloudFormation already
  understands the ordering from the reference alone.

### Cleanup

```bash
aws cloudformation delete-stack --stack-name hello-cognito --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name hello-cognito --region us-east-1
```

---

## Git sync — automated deploys from GitHub

`deployments/deployment-main.yaml` is a CloudFormation Git sync deployment
file. It tells CloudFormation which template to deploy and what parameter
values to pass.

### Recommended repo layout for Git sync

```
your-repo/
├── templates/
│   └── orchestrator-stack.yaml      ← or whichever template you're syncing
└── deployment-main.yaml             ← the deployment file
```

### Quick start

1. Push your template + `deployment-main.yaml` to a GitHub branch (typically `main`).
2. AWS Console → Developer Tools → Settings → Connections → create a GitHub
   connection (installs the **AWS Connector for GitHub** GitHub App).
3. CloudFormation console → Create stack → **Sync from Git** → select the
   connection, repo, branch, and path to `deployment-main.yaml`.
4. Subsequent commits to the watched branch auto-deploy.

Full step-by-step instructions including troubleshooting are in
`docs/git-sync-guide.md`.

### Common gotchas (all hit during this session)

| Error | Cause | Fix |
|---|---|---|
| `Unable to read CloudFormation Deployment Config File` | Leading YAML comments confused the parser | Strip comments or use the minimal form |
| `Resource not found for request with filepath: ...` | `template-file-path` points to a file that doesn't exist on the watched branch | Match the path exactly, case-sensitive, no leading slash |
| `Template format error: YAML not well-formed. (line 1, column 23)` | BOM character or smart quotes from a copy-paste | Recreate the file via GitHub's web UI, or save as UTF-8 without BOM |
| `Failed to create changeset` referencing "execution roles" | Stack execution role doesn't have IAM perms for the new resource types | Attach `AdministratorAccess` (dev) or add scoped Cognito + IAM perms (prod) |
| `Invalid request provided: AWS::Cognito::UserPoolDomain` | Domain prefix contains a reserved word (`aws`, `amazon`, `cognito`) | Rename the stack and/or `AppName` to avoid these substrings |

---

## A note on SAM

During this session you also ran `sam init`, which scaffolds a Serverless
Application Model project. SAM is a CloudFormation dialect optimized for
serverless apps — a more compact `AWS::Serverless::Function` expands into
the Lambda + role + API Gateway + permission combo we wrote by hand here.

The two approaches are not in conflict — many teams use plain CloudFormation
for general infrastructure and SAM for the serverless app layer. The bundle
in this folder is plain CloudFormation; if you want to translate, the
deployable result is identical, just less verbose.

---

## Reading order

For someone new to CloudFormation:

1. Read `templates/02-hello-world-site-commented.yaml` end to end — it explains
   every intrinsic function and every API Gateway requirement.
2. Deploy `01-hello-world-site.yaml` and hit the URL.
3. Deploy `03-orchestrator-stack.yaml`, invoke the orchestrator, and watch
   the processor's logs.
4. Edit the VPC tag on the orchestrator template, redeploy, look at the
   changeset — that's your "infrastructure as code" intuition forming.
5. Deploy `04-hello-world-cognito.yaml` and walk through the full auth flow.
6. Set up Git sync per `docs/git-sync-guide.md` and observe the automated
   deploys when you commit changes.

---

## Cleanup

When you're done, delete all stacks to avoid lingering costs (most of these
resources are free-tier, but Cognito user pools and S3 buckets can accumulate
if forgotten):

```bash
aws cloudformation delete-stack --stack-name hello-world-site     --region us-east-1
aws cloudformation delete-stack --stack-name orchestrator-demo    --region us-east-1
aws cloudformation delete-stack --stack-name hello-cognito        --region us-east-1
```

The orchestrator stack's *dynamically created* S3 buckets won't be deleted by
the stack (CloudFormation didn't create them — the Lambda did). Clean those
up separately:

```bash
# List buckets the orchestrator created
aws s3 ls | grep synthetic-data-

# Delete each one (empty first if needed)
aws s3 rb s3://synthetic-data-xxxxxx --force
```
