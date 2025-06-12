Here’s a complete solution to implement drift monitoring for Kubernetes applications in Spinnaker, using a GitOps-based approach to ensure the live state of the Kubernetes cluster matches the desired state defined in Git.

✅ Goal
Detect and alert if the actual Kubernetes resources in the cluster drift from the Git-based configuration managed via Spinnaker CD pipelines.

🔧 Solution Architecture
lua
Copy
Edit
         +------------------+
         |  Git Repository  |
         | (Desired State)  |
         +--------+---------+
                  |
                  | Git Pull
         +--------v---------+
         | Drift Detector   |   <-- (Script/CronJob/Spinnaker stage)
         +--------+---------+
                  |
        +---------v--------+
        | kubectl / diff   |   <-- Compare actual vs desired
        +---------+--------+
                  |
        +---------v--------+
        | Notification /   |
        |  Manual Judgment |
        +------------------+
🛠️ Implementation Steps
Step 1: Define the Desired State (Git Source)
Store your Kubernetes manifests in a Git repository. These should reflect the intended deployment state.

Example folder structure:

markdown
Copy
Edit
k8s-manifests/
├── production/
│   └── app-deployment.yaml
└── staging/
    └── app-deployment.yaml
Step 2: Create Drift Detection Script
This script fetches the latest Git repo, compares the live deployment in the cluster, and reports differences.

✅ drift-check.sh
bash
Copy
Edit
#!/bin/bash

set -e

# Clone manifests repo (or pull latest)
REPO_URL="https://github.com/your-org/k8s-manifests.git"
NAMESPACE="production"
APP_NAME="app-deployment"
WORK_DIR="/tmp/k8s-manifests"

rm -rf $WORK_DIR
git clone $REPO_URL $WORK_DIR

# Get live state
kubectl get -n $NAMESPACE deployment $APP_NAME -o yaml > /tmp/live.yaml

# Get desired state from Git
cp $WORK_DIR/$NAMESPACE/${APP_NAME}.yaml /tmp/desired.yaml

# Compare
diff_output=$(diff -u /tmp/desired.yaml /tmp/live.yaml || true)

if [ -n "$diff_output" ]; then
  echo "🚨 Drift detected!"
  echo "$diff_output"
  echo "$diff_output" | mail -s "Drift Alert: $APP_NAME" devops@example.com
  exit 1
else
  echo "✅ No drift detected"
fi
Step 3: Run the Script in Spinnaker Pipeline
Create a Spinnaker pipeline with a Run Job or Script stage that executes the drift-check.sh.

Example: Spinnaker “Run Job” Stage
Use a Kubernetes Job like this in the pipeline:

yaml
Copy
Edit
apiVersion: batch/v1
kind: Job
metadata:
  name: drift-check-job
spec:
  template:
    spec:
      containers:
      - name: drift-check
        image: bitnami/kubectl:latest
        command: ["/bin/bash", "-c"]
        args:
          - apt update && apt install -y git mailutils &&
            chmod +x /scripts/drift-check.sh &&
            /scripts/drift-check.sh
        volumeMounts:
        - name: script-volume
          mountPath: /scripts
      restartPolicy: Never
      volumes:
      - name: script-volume
        configMap:
          name: drift-check-script
Create the ConfigMap:

bash
Copy
Edit
kubectl create configmap drift-check-script --from-file=drift-check.sh
Step 4: Trigger Pipeline Periodically
Use Spinnaker’s Cron trigger to run this pipeline every 30 minutes or 1 hour.

Step 5: Add Manual Judgment (Optional)
If drift is detected, insert a Manual Judgment stage in Spinnaker:

Option A: Approve and redeploy the correct manifests

Option B: Ignore if the drift is intentional (e.g., hotfix)

✅ Output / Results
🚨 If drift is detected → Email/Slack alert + Pipeline stops

✅ If no drift → Pipeline succeeds silently

📒 All comparisons and alerts are logged

🔄 Optional Enhancements
Feature	Description
🔄 Auto-remediation	Automatically reapply manifests using kubectl apply or Spinnaker deploy stage
🔐 RBAC Check	Restrict cluster edits outside of pipeline
📊 Drift Dashboard	Visualize drift metrics using Prometheus/Grafana
🔍 Audit Trail	Store diff logs for compliance and analysis
