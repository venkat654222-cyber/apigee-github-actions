Apigee X CI/CD Integration –Architecture Overview and Security Model
Overview
This CI/CD implementation integrates Apigee X with GitHub Actions using apigeecli to automate the deployment of API proxies, shared flows, key-value maps (KVMs), and target servers.
The solution follows infrastructure-as-code and DevOps best practices, where:
1.	Apigee artifacts are stored and versioned in GitHub
2.	GitHub Actions acts as the CI/CD execution engine
3.	Apigee CLI (apigeecli) performs deployments programmatically
4.	Google Cloud IAM provides secure, auditable authentication and authorization
This approach eliminates manual deployments through the Apigee UI, ensuring consistent, repeatable, and secure releases across environments.
Note: In this document and setup, bamboo-zone-363116 and 1041995087153 are used as example Apigee organization / project IDs. The service account apigee-deploy-sa@bamboo-zone-363116.iam.gserviceaccount.com is provided as an example. The eval environment is used as the Apigee environment for testing deployments.

1.1	Role of apigeecli in the CI/CD Pipeline
apigeecli is the official command-line interface used to interact with Apigee X programmatically.
In this CI/CD pipeline, apigeecli is responsible for:
a)	Deploying API proxies and shared flows
b)	Creating and updating KVMs
c)	Importing target servers
d)	By using apigeecli inside GitHub Actions:
e)	All deployments are automated
f)	Configuration drift is minimized
g)	Changes are traceable through Git history
h)	Rollbacks can be performed easily

1.2	Authentication Model- OIDC with Workload Identity Federation
What is OIDC in this context?
OIDC (OpenID Connect) is used to authenticate GitHub Actions to Google Cloud without storing any long-lived credentials.
In this architecture:
•	GitHub Actions generates a short-lived OIDC token at runtime
•	Google Cloud IAM validates this token via Workload Identity Federation
•	The token is exchanged for temporary credentials
•	The GitHub workflow impersonates a Google Cloud service account
•	apigeecli uses these temporary credentials to deploy to Apigee X

Why OIDC Is Used Instead of Service Account JSON Keys
Traditional approach (NOT recommended), CI/CD pipelines authenticate using a service account JSON key stored in GitHub Secrets.
This approach has several drawbacks:
•	JSON keys are long-lived credentials
•	If leaked, they grant full access until manually revoked
•	Keys must be rotated periodically
•	Secrets can be accidentally exposed in logs or misconfigured workflows

OIDC-based approach (RECOMMENDED)
Using OIDC with Workload Identity Federation provides major security and operational advantages:
No stored secrets
•	No service account JSON keys are stored in GitHub
•	Nothing sensitive is checked into repositories or secrets
 Short-lived credentials
•	Tokens are valid only for minutes
•	Automatically expire after workflow execution
 Least privilege access
•	Access can be restricted to:
o	A specific GitHub repository
o	A specific repository owner
o	A specific branch or workflow (optional)
Improved security posture
•	Eliminates risk of credential leakage
•	Prevents reuse of stolen credentials
•	Reduces attack surface
 Auditability
•	All access is logged in Google Cloud Audit Logs
•	Clear traceability between GitHub workflow runs and GCP actions

High-Level Authentication Flow
1.	A GitHub Actions workflow starts
2.	GitHub issues an OIDC token for the workflow
3.	Google Cloud validates the token via Workload Identity Pool
4.	The token is mapped to a trusted identity
5.	The workflow impersonates a Google Cloud service account
6.	apigeecli uses temporary credentials
7.	Apigee X resources are deployed securely

Summary
This CI/CD architecture:
•	Integrates Apigee X and GitHub Actions
•	Uses apigeecli for automation
•	Implements OIDC-based Workload Identity Federation
•	Eliminates long-lived secrets
•	Aligns with Google Cloud and DevOps security best practices
This model is more secure, scalable, and production-ready than traditional key-based authentication methods.
2.	GitHub repository
To verify the Repo structure and GitHub actions YAML file, please refer path below.
https://github.com/venkat654222-cyber/apigee-github-actions

3.	Configuration Steps
3.1	Enabling IAM Service Account Credentials API in gcoud console
Why this is required
This API allows:
•	Short-lived credential generation
•	Service account impersonation
Without it:
❌ Workload Identity Federation will fail.
Log in into gcloud cosole and enable the api as per the screenshot below.

  
3.2	Creating a Service Account in gcloud console
Why is this required?
A service account represents a non-human identity that GitHub Actions can use to deploy resources into Apigee.
Instead of using personal credentials or long-lived keys, the CI/CD pipeline authenticates securely using this service account.

a)	Search for Service Accounts in google cloud console
This allows you to manage identities used by applications and automation instead of human users.

b)	Click Create Service Account
Creates a new identity dedicated to Apigee CI/CD operations.
Enter a name as apigee-deploy-sa(example)
c)	Assign roles while creating the service account

Role: Apigee Environment Admin
•	Allows deployment of API proxies
•	Allows deployment of shared flows
•	Manages Apigee environment resources

Role: Apigee API Admin
•	Allows creation and update of API proxies
•	Manages proxy revisions and deployments

Click Create and Continue
3.3	Setting Up GitHub Actions Secrets
Navigate to settings in GitHub repo

 
Add APIGEE_ORG as a name and value as bamboo-zone-363116(org id/project id)
Why this is required
•	Keeps sensitive configuration out of source code
•	Makes the pipeline reusable across environments

3.4	Creating a Workload Identity Pool in gcloud console
gcloud iam workload-identity-pools create "github-pool" \
  --project="bamboo-zone-363116" \
  --location="global" \
  --display-name="GitHub Actions Pool"
Why this is required
A Workload Identity Pool allows external identities (GitHub) to authenticate with Google Cloud securely using OIDC tokens instead of keys.

3.5	Creating a GitHub OIDC Provider in gcloud console
gcloud iam workload-identity-pools providers create-oidc "github" \
  --project="bamboo-zone-363116" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Provider" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository,attribute.repository_owner=assertion.repository_owner" \
  --attribute-condition="assertion.repository_owner == 'bamboo-zone-363116'"
Why this is required
This step establishes trust between:
•	GitHub Actions
•	Google Cloud IAM
It ensures only approved repositories can authenticate.

3.6	Retrieving the Provider Resource Name in gcloud console
gcloud iam workload-identity-pools providers describe "github" \
  --project="bamboo-zone-363116" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --format="value(name)"
Why this is required
The provider resource name is needed when binding IAM permissions between GitHub and the service account.

3.7	Retrieving the Project Number in gcloud console
gcloud projects describe bamboo-zone-363116 \
  --format="value(projectNumber)"
Why this is required
IAM bindings for workload identity use the project number, not the project ID.

3.8	Allow GitHub to Impersonate the Service Account in gcloud console
gcloud iam service-accounts add-iam-policy-binding \
  apigee-deploy-sa@bamboo-zone-363116.iam.gserviceaccount.com \
  --project="bamboo-zone-363116" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/1041995087153/locations/global/workloadIdentityPools/github-pool/attribute.repository/venkat654222-cyber/apigee-github-actions"
Why this is required
This explicitly allows only the specified GitHub repository to impersonate the service account.
Without this binding:
❌ GitHub Actions cannot authenticate to Google Cloud.

3.9	Restricting Access by Repository Owner in gcloud console
gcloud iam workload-identity-pools providers update-oidc "github" \
  --project="bamboo-zone-363116" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --attribute-condition="assertion.repository_owner == 'venkat654222-cyber'"
Why this is required
Adds an additional security layer to prevent unauthorized repositories or forks from accessing GCP.
3.10	Reconfirming Apigee Permissions in gcloud console
gcloud projects add-iam-policy-binding "bamboo-zone-363116" \
  --member="serviceAccount:apigee-deploy-sa@bamboo-zone-363116.iam.gserviceaccount.com" \
  --role="roles/apigee.environmentAdmin"
Why this is required
Ensures the service account always has permission to deploy Apigee resources, even if IAM policies change later.
4.	Apigee KVMs
Create Key Value Maps (KVMs) json fie in GitHub repo
Why this is required
Key Value Maps (KVMs) and Target Servers are core runtime configuration components in Apigee X.
They define environment-specific values such as:
•	Backend URLs
•	Credentials and tokens
•	Timeouts and configuration flags
To achieve a reliable CI/CD process, GitHub must act as the single source of truth for these configurations, instead of manual changes in the Apigee UI.
Example GitHub repo path for KVM and sample .json file
GitHub\apigee-github-actions\configs\dev\kvms\backEndCreds.json
{
    "name": "backEndCreds",
    "encrypted": true,
    "entry": [
      { "name": "username1", "value": "sap" },
      { "name": "password", "value": "1234556" }
    ]
  }
5.	Apigee Target Servers
Example GitHub repo path for target Server and sample .json file
GitHub\apigee-github-actions\configs\dev\kvms\oauth-server-ts.json
[
    {
      "name": "oauth-server-ts",
      "host": "api.oauth-provider.com",
      "port": 443,
      "isEnabled": true,
      "protocol": "HTTP"
    }
  ]

6.	Apigee Shared Flows 
 Commit shared flow code as shown in the screenshot below.
 
7.	Apigee API Proxies
Commit the API Proxy code as per the screenshot below.
 
7.1	Build.yaml
name: Apigee X CI/CD for KVMs, Target Servers, Shared flows and API Proxies

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/1041995087153/locations/global/workloadIdentityPools/github-pool/providers/github
          service_account: apigee-deploy-sa@bamboo-zone-363116.iam.gserviceaccount.com

      - name: Set Apigee Environment
        run: |
          if [[ "${{ github.ref_name }}" == "dev" ]]; then
            echo "APIGEE_ENV=dev" >> $GITHUB_ENV
          elif [[ "${{ github.ref_name }}" == "test" ]]; then
            echo "APIGEE_ENV=test" >> $GITHUB_ENV
          elif [[ "${{ github.ref_name }}" == "main" ]]; then
            echo "APIGEE_ENV=eval" >> $GITHUB_ENV
          else
            echo "Unsupported branch: ${{ github.ref_name }}"
            exit 1
          fi

      - name: Install apigeecli
        run: |
          curl -L https://raw.githubusercontent.com/apigee/apigeecli/main/downloadLatest.sh -o install_apigeecli.sh
          if [ -s install_apigeecli.sh ]; then
            chmod +x install_apigeecli.sh
            ./install_apigeecli.sh
            echo "$HOME/.apigeecli/bin" >> $GITHUB_PATH
            echo "✅ apigeecli installed successfully."
          else
            echo "❌ Error: apigeecli installer download failed."
            exit 1
          fi


               

      - name: Deploy KVMs
        run: |
          if [ -d "configs/${{ env.APIGEE_ENV }}/kvms" ] && [ "$(ls -A configs/${{ env.APIGEE_ENV }}/kvms/*.json 2>/dev/null)" ]; then
            for file in configs/${{ env.APIGEE_ENV }}/kvms/*.json; do
              KVM_NAME=$(basename "$file" .json)
              echo "Processing KVM: $KVM_NAME"
              apigeecli kvms create \
                --org "${{ secrets.APIGEE_ORG }}" \
                --env "${{ env.APIGEE_ENV }}" \
                --name "$KVM_NAME" \
                --default-token || true
              apigeecli kvms entries import \
                --org "${{ secrets.APIGEE_ORG }}" \
                --env "${{ env.APIGEE_ENV }}" \
                --map "$KVM_NAME" \
                -f "$file" \
                --default-token
              echo "✅ KVM '$KVM_NAME' synced successfully."
            done
          else
            echo "ℹ️ No KVM configurations found to deploy."
          fi

      - name: Deploy Modified Target Servers
        run: |
          MODIFIED_TS=$(git diff --name-only HEAD^ HEAD | grep "^configs/${{ env.APIGEE_ENV }}/target-servers/" | sort -u || true)
          if [ -n "$MODIFIED_TS" ]; then
            for file in $MODIFIED_TS; do
              TS_NAME=$(basename "$file" .json)
              echo "🚀 Syncing Modified Target Server: $TS_NAME"
              apigeecli targetservers import \
                --org "${{ secrets.APIGEE_ORG }}" \
                --env "${{ env.APIGEE_ENV }}" \
                -f "$file" \
                --default-token
              echo "✅ Target Server '$TS_NAME' deployed successfully."
            done
          else
            echo "ℹ️ No Target Servers modified in this commit."
          fi

      - name: Deploy Modified Shared Flows
        run: |
          MODIFIED_SFS=$(git diff --name-only HEAD^ HEAD | grep "^sharedflows/${{ env.APIGEE_ENV }}/" | cut -d/ -f3 | sort -u || true)
          if [ -n "$MODIFIED_SFS" ]; then
            for SF_NAME in $MODIFIED_SFS; do
              if [ -d "sharedflows/${{ env.APIGEE_ENV }}/$SF_NAME/sharedflowbundle" ]; then
                echo "🚀 Creating and Deploying Shared Flow: $SF_NAME"
                apigeecli sharedflows create bundle \
                  --name "$SF_NAME" \
                  --sf-folder "sharedflows/${{ env.APIGEE_ENV }}/$SF_NAME/sharedflowbundle" \
                  --org "${{ secrets.APIGEE_ORG }}" \
                  --default-token
                apigeecli sharedflows deploy \
                  --name "$SF_NAME" \
                  --org "${{ secrets.APIGEE_ORG }}" \
                  --env "${{ env.APIGEE_ENV }}" \
                  --ovr \
                  --default-token
                echo "✅ Shared Flow '$SF_NAME' deployed successfully."
              fi
            done
          else
            echo "ℹ️ No Shared Flows modified in this commit."
          fi

      - name: Deploy Modified API Proxies
        run: |
          MODIFIED_PROXIES=$(git diff --name-only HEAD^ HEAD | grep "^proxies/${{ env.APIGEE_ENV }}/" | cut -d/ -f3 | sort -u || true)
          if [ -n "$MODIFIED_PROXIES" ]; then
            for PROXY_NAME in $MODIFIED_PROXIES; do
              if [ -d "proxies/${{ env.APIGEE_ENV }}/$PROXY_NAME/apiproxy" ]; then
                echo "🚀 Creating and Deploying Proxy: $PROXY_NAME"
                apigeecli apis create bundle \
                  --name "$PROXY_NAME" \
                  --proxy-folder "proxies/${{ env.APIGEE_ENV }}/$PROXY_NAME/apiproxy" \
                  --org "${{ secrets.APIGEE_ORG }}" \
                  --default-token
                apigeecli apis deploy \
                  --name "$PROXY_NAME" \
                  --org "${{ secrets.APIGEE_ORG }}" \
                  --env "${{ env.APIGEE_ENV }}" \
                  --ovr \
                  --default-token
                echo "✅ API Proxy '$PROXY_NAME' deployed successfully."
              fi
            done
          else
            echo "ℹ️ No API Proxies modified in this commit."
          Fi

7.2	Git Repo structure

apigee-github-actions/
│
├── .github/
│   └── workflows/
│       └── build.yaml
│
├── proxies/
│   ├── dev/
│   │   └── sample-api/
│   │       └── apiproxy/
│   ├── test/
│   ├── eval/
│   └── prod/
│
├── sharedflows/
│   ├── dev/
│   │   └── common-security/
│   │       └── sharedflowbundle/
│   ├── test/
│   ├── eval/
│   └── prod/
│
├── configs/
│   ├── dev/
│   │   ├── kvms/
│   │   ├── target-servers/
│   │   └── keystores/
│   │       └── keystores.json
│   ├── test/
│   ├── eval/
│   └── prod/
│
├── README.md
└── docs/
    └── apigee-ci-cd-design.docx
7.3	Example Mapping GitHub branch folder-Apigee env
GitHub Repository Folder	Apigee Target Environment
proxies/dev/	Apigee dev
proxies/test/	Apigee test
proxies/eval/	Apigee eval
proxies/prod/	Apigee prod
configs/dev/	Apigee dev
sharedflows/dev/	Apigee dev
