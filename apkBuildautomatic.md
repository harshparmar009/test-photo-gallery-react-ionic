# Automated Android APK Build Pipeline

## Objective

The objective of this implementation is to automate Android APK generation for an Ionic Capacitor application whenever code changes are pushed to the GitHub repository.

This eliminates manual APK generation and provides a fully automated CI/CD workflow using Argo Events and Argo Workflows running inside Kubernetes.

---

# Architecture Overview

```text
Developer
    │
    ▼
GitHub Repository
    │
    ▼
GitHub Webhook
    │
    ▼
ngrok Public Endpoint
    │
    ▼
Argo Events EventSource
    │
    ▼
Argo Events Sensor
    │
    ▼
Argo Workflow
    │
    ▼
Build APK
    │
    ▼
MinIO Artifact Storage
    │
    ▼
APK Download
```

---

# Components Used

| Component         | Purpose                                      |
| ----------------- | -------------------------------------------- |
| Kubernetes        | Container orchestration platform             |
| Argo Workflows    | Workflow execution engine                    |
| Argo Events       | Event-driven workflow triggering             |
| GitHub Webhooks   | Trigger workflow on code push                |
| ngrok             | Public endpoint for local Kubernetes cluster |
| MinIO             | Artifact storage for generated APK files     |
| Docker            | Android build environment                    |
| Ionic + Capacitor | Mobile application framework                 |

---

# Workflow Process

## Step 1: Developer Pushes Code

Developer commits and pushes code changes to GitHub.

```bash
git add .
git commit -m "Updated application"
git push
```

---

## Step 2: GitHub Webhook Trigger

GitHub sends a webhook event whenever code is pushed to the repository.

Repository:

```text
https://github.com/harshparmar009/test-photo-gallery-react-ionic
```

Webhook Payload:

```text
Push Event
Branch: main
```

---

## Step 3: Argo Events Receives Event

The EventSource receives the GitHub webhook request through the public ngrok endpoint.

EventSource Endpoint:

```text
/github
```

---

## Step 4: Sensor Validates Event

The Sensor validates:

* Repository name
* Branch name
* Event type

After validation, the Sensor automatically triggers the workflow.

---

## Step 5: Workflow Execution

Argo Workflow creates a new workflow instance.

Example:

```text
ionic-apk-auto-gstft
```

Workflow Status:

```text
Running
Succeeded
Failed
```

---

# APK Build Process

The workflow performs the following operations:

## Clone Repository

```bash
git clone https://github.com/harshparmar009/test-photo-gallery-react-ionic.git
```

## Install Dependencies

```bash
npm ci
```

## Build Ionic Application

```bash
npm run build
```

## Capacitor Android Synchronization

```bash
npx cap sync android
```

## Android APK Generation

```bash
./gradlew assembleDebug
```

---

# Build Output

Generated APK:

```text
app-debug.apk
```

Workflow Log Example:

```text
BUILD SUCCESSFUL
APK build completed
```

---

# Artifact Storage

After successful APK generation:

```text
APK
 ↓
Compress (.tgz)
 ↓
Upload to MinIO
 ↓
Store as Workflow Artifact
```

Artifact Example:

```text
ionic-apk.tgz
```

Artifact Location:

```text
MinIO Bucket: argo-artifacts
```

Workflow Output:

```text
Artifact Name: ionic-apk
```

---

# APK Download Process

1. Open Argo Workflows UI
2. Select successful workflow
3. Open workflow details
4. Navigate to Artifacts
5. Download artifact

Downloaded File:

```text
ionic-apk.tgz
```

Extract:

```text
ionic-app-debug.apk
```

Install APK on Android device.

---

# Workflow Validation

Successful workflow execution is verified using:

```bash
kubectl get workflows -n argo
```

Example Output:

```text
NAME                   STATUS
ionic-apk-auto-gstft   Succeeded
```

---

# Benefits

## Automation

No manual APK build required.

## Faster Delivery

APK generated immediately after code push.

## Repeatability

Consistent build environment for every build.

## Traceability

Every build is linked to a GitHub commit and workflow execution.

## Scalability

Supports multiple projects and parallel workflow execution.

---

# Result

A complete event-driven CI/CD pipeline was implemented using Argo Events and Argo Workflows.

Whenever code is pushed to GitHub:

1. Webhook triggers automatically.
2. Argo Events receives the event.
3. Argo Workflow starts automatically.
4. Ionic application is built.
5. Android APK is generated.
6. APK is stored as an artifact in MinIO.
7. APK becomes available for download through Argo Workflows UI.

No manual workflow execution or APK generation is required.


*************************************************************************************************************************************

Order of Starting the CI/CD Pipeline


1. Start Docker Desktop

Open Docker Desktop and wait until:

Engine running
Kubernetes running

Verify:

kubectl get nodes

Expected:

desktop-control-plane   Ready


2. Check Pods
Argo namespace
kubectl get pods -n argo

Expected:

argo-server
workflow-controller
minio
github-eventsource
github-apk-sensor
eventbus
Argo Events namespace
kubectl get pods -n argo-events

Expected:

controller-manager   Running


3. Start ngrok

Open a new terminal:

ngrok http 12000

Copy the forwarding URL.

If it changes (free plan), update the GitHub webhook:

GitHub
→ Settings
→ Webhooks
→ Edit
→ Payload URL

https://xxxxx.ngrok-free.dev/github


4. Port-forward the EventSource

Open another terminal:

kubectl port-forward svc/github-eventsource-eventsource-svc -n argo 12000:12000

Keep this terminal open.


5. Start Argo UI

Open another terminal:

kubectl port-forward svc/argo-server -n argo 2746:2746

Open:

https://localhost:2746


6. (Optional) Start MinIO

If you want to download APK artifacts:

kubectl port-forward svc/minio -n argo 9000:9000


7. Verify Everything

Watch new workflow create
kubectl get workflows -n argo -w

kubectl get pods -n argo
kubectl get pods -n argo-events

Everything should be Running (completed workflow pods are expected to show Completed).


8. Test the Pipeline

Make a code change:

git add .
git commit -m "Test"
git push


Watch:
cmd - kubectl get workflows -n argo -w
A new workflow should appear automatically.

Watch APK build log
cmd - kubectl logs -n argo <your-ionic-apk-pod> -f





Daily Startup Checklist
✅ Start Docker Desktop
✅ Wait for Kubernetes
✅ kubectl get nodes

✅ kubectl get pods -n argo
✅ kubectl get pods -n argo-events

✅ Terminal 1:
kubectl port-forward svc/github-eventsource-eventsource-svc -n argo 12000:12000

✅ Terminal 2:
ngrok http 12000

✅ Update GitHub webhook if ngrok URL changed

✅ Terminal 3:
kubectl port-forward svc/argo-server -n argo 2746:2746

✅ Open https://localhost:2746

✅ Push code and verify a new workflow starts