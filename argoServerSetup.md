# Complete CI/CD Setup Documentation

## Ionic App Deployment with Argo CD + Automatic Android APK Build using Argo Events and Argo Workflows

---

## 1. Objective

The objective of this setup is to create a complete CI/CD pipeline for an Ionic Capacitor project.

This setup includes:

1. Web application deployment using Docker, Kubernetes, and Argo CD.
2. Automatic Android APK generation using Argo Events and Argo Workflows.
3. Artifact storage using MinIO.
4. GitHub webhook trigger for automatic APK build after code push.

Final result:

```text
Developer pushes code to GitHub
        ↓
GitHub Webhook triggers Argo Events
        ↓
Argo Sensor starts Argo Workflow
        ↓
Workflow clones latest Ionic code
        ↓
Ionic app builds
        ↓
Capacitor syncs Android
        ↓
Gradle builds APK
        ↓
APK is stored in MinIO artifact storage
        ↓
APK is downloadable from Argo Workflows UI
```

---

# 2. Tools Used

```text
Docker Desktop
Kubernetes
kubectl
GitHub
Docker Hub
Argo CD
Argo Workflows
Argo Events
ngrok
MinIO
Ionic React
Capacitor
Gradle
Android SDK
```

---

# 3. Project Repository i useed

```text
GitHub Repo:
https://github.com/harshparmar009/test-photo-gallery-react-ionic
```

---

# 4. Folder Structure

Final project structure:

```text
test-photo-gallery-react-ionic/
│
├── src/
├── public/
├── android/
├── package.json
├── capacitor.config.ts
├── Dockerfile
├── .dockerignore
│
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
│
├── apk-builder/
│   ├── Dockerfile.ionic-android
│   └── .dockerignore
│
└── argo/
    ├── workflow-rbac.yaml
    ├── minio.yaml
    ├── apk-workflow-template.yaml
    ├── run-apk-workflow.yaml
    ├── eventbus.yaml
    ├── eventsource.yaml
    ├── sensor-rbac.yaml
    └── sensor.yaml
```

---

# 5. Prerequisites

Install:

```text
Docker Desktop
Node.js
Git
kubectl
ngrok
```

Verify:

```bash
docker version
kubectl version --client
git --version
node -v
npm -v
```

Enable Kubernetes in Docker Desktop:

```text
Docker Desktop
→ Settings
→ Kubernetes
→ Enable Kubernetes
```

Check Kubernetes:

```bash
kubectl get nodes
```

Expected:

```text
desktop-control-plane   Ready
```

---

# 6. Clone Ionic Project

```bash
git clone https://github.com/harshparmar009/test-photo-gallery-react-ionic.git
cd test-photo-gallery-react-ionic
```

Install dependencies:

```bash
npm install
```

Build project:

```bash
npm run build
```

Sync Android:

```bash
npx cap sync android
```

If Android folder is missing:

```bash
npx cap add android
npx cap sync android
```

---

# 7. Dockerize Ionic Web App

Create `Dockerfile`:

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Create `.dockerignore`:

```text
node_modules
dist
.git
android/.gradle
android/app/build
ios
```

Build image:

```bash
docker build -t harshparmar009/ionic-gallery:v1 .
```

Run locally:

```bash
docker run -d -p 8080:80 --name ionic-test harshparmar009/ionic-gallery:v1
```


Open:

```text
http://localhost:8080
```

Stop container:

```bash
docker stop ionic-test
docker rm ionic-test
```

Push to Docker Hub:

```bash
docker push harshparmar009/ionic-gallery:v1
```

---

# 8. Kubernetes Manifests

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ionic-gallery
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ionic-gallery
  template:
    metadata:
      labels:
        app: ionic-gallery
    spec:
      containers:
        - name: ionic-gallery
          image: harshparmar009/ionic-gallery:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```

Create `k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ionic-gallery-service
spec:
  selector:
    app: ionic-gallery
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

Create `k8s/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ionic-gallery-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: ionic.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ionic-gallery-service
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f k8s/
```

Check:

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

Hosts file:

open Notebook app in run as administration and open

```text
C:\Windows\System32\drivers\etc\hosts
```

Add: At Bottom

```text
127.0.0.1 ionic.local
```

Open:

```text
http://ionic.local
```

---

# 9. Argo CD Installation

Create namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check pods:

```bash
kubectl get pods -n argocd
```

Port forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Get admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"
```

Decode in PowerShell:

```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("<PASTE_PASSWORD>"))
```

Create Argo CD application:

```text
Application Name: ionic-gallery
Repo URL: https://github.com/harshparmar009/test-photo-gallery-react-ionic
Path: k8s
Cluster: https://kubernetes.default.svc
Namespace: default
Sync Policy: Automatic
```

Expected result:

```text
Synced
Healthy
```

---

# 10. Web App Update Flow

Whenever code changes for the web app:

```bash
docker build -t harshparmar009/ionic-gallery:v2 .
docker push harshparmar009/ionic-gallery:v2
```

Update `k8s/deployment.yaml`:

```yaml
image: harshparmar009/ionic-gallery:v2
```

Push changes:

```bash
git add .
git commit -m "Update web app image to v2"
git push
```

Argo CD automatically syncs Kubernetes.

---

# 11. Argo Workflows Installation

Create namespace:

```bash
kubectl create namespace argo
```

Install Argo Workflows:

```bash
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.12/install.yaml
```

Check:

```bash
kubectl get pods -n argo
```

Expected:

```text
argo-server             Running
workflow-controller     Running
```

Issue faced:

```text
workflow-controller CrashLoopBackOff
error: workflows.argoproj.io not found
```

Reason:

```text
Argo Workflows CRDs were missing.
```

Fix:

```bash
kubectl delete namespace argo
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.12/install.yaml
```

---

# 12. Workflow RBAC

Create `argo/workflow-rbac.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workflow
  namespace: argo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: workflow-role
  namespace: argo
rules:
  - apiGroups: ["argoproj.io"]
    resources:
      - workflows
      - workflowtaskresults
    verbs: ["get", "watch", "patch", "create"]
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: workflow-binding
  namespace: argo
subjects:
  - kind: ServiceAccount
    name: workflow
    namespace: argo
roleRef:
  kind: Role
  name: workflow-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f argo/workflow-rbac.yaml
```

Issue faced:

```text
workflowtaskresults.argoproj.io is forbidden
```

Reason:

```text
Workflow pod was using default service account without permission.
```

Fix:

```text
Created workflow service account and assigned permissions.
```

---

# 13. Test Argo Workflow

Create `argo/test-workflow.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-
  namespace: argo
spec:
  serviceAccountName: workflow
  entrypoint: hello
  templates:
    - name: hello
      container:
        image: alpine
        command: [sh, -c]
        args:
          - echo "Hello from Argo Workflows"
```

Run:

```bash
kubectl create -f argo/test-workflow.yaml
```

Check:

```bash
kubectl get workflows -n argo
```

Logs:

```bash
kubectl logs -n argo <hello-pod-name>
```

Expected:

```text
Hello from Argo Workflows
```

---

# 14. Android Builder Docker Image

Create folder:

```bash
mkdir apk-builder
```

Create `apk-builder/Dockerfile.ionic-android`:

```dockerfile
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV ANDROID_HOME=/opt/android-sdk
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
ENV PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools

RUN apt-get update && apt-get install -y \
    openjdk-17-jdk wget curl git unzip build-essential python3 \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get install -y nodejs && \
    npm install -g npm@latest @ionic/cli

RUN mkdir -p $ANDROID_HOME/cmdline-tools && \
    wget -q https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip \
    -O /tmp/cmdtools.zip && \
    unzip -q /tmp/cmdtools.zip -d $ANDROID_HOME/cmdline-tools && \
    mv $ANDROID_HOME/cmdline-tools/cmdline-tools $ANDROID_HOME/cmdline-tools/latest && \
    rm /tmp/cmdtools.zip

RUN yes | sdkmanager --licenses && \
    sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0"

WORKDIR /workspace
```

Create `apk-builder/.dockerignore`:

```text
.git
node_modules
dist
android
ios
*.log
```

Build:

```bash
cd apk-builder
docker build --no-cache -f Dockerfile.ionic-android -t harshparmar009/ionic-android-builder:v1 .
docker push harshparmar009/ionic-android-builder:v1
cd ..
```

Note:

```text
This image is not the app image.
It is only the build environment for APK generation.
```

---

# 15. MinIO Artifact Storage

Create `argo/minio.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio
  namespace: argo
type: Opaque
stringData:
  accesskey: minio
  secretkey: minio123
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: argo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: minio/minio:latest
          args: ["server", "/data"]
          env:
            - name: MINIO_ROOT_USER
              value: minio
            - name: MINIO_ROOT_PASSWORD
              value: minio123
          ports:
            - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: argo
spec:
  selector:
    app: minio
  ports:
    - port: 9000
      targetPort: 9000
```

Apply:

```bash
kubectl apply -f argo/minio.yaml
```

Check:

```bash
kubectl get pods -n argo
kubectl get svc -n argo
```

Edit workflow controller config:

```bash
kubectl edit configmap workflow-controller-configmap -n argo
```

Add:

```yaml
data:
  artifactRepository: |
    s3:
      bucket: argo-artifacts
      endpoint: minio.argo:9000
      insecure: true
      accessKeySecret:
        name: minio
        key: accesskey
      secretKeySecret:
        name: minio
        key: secretkey
```

Restart:

```bash
kubectl rollout restart deployment workflow-controller -n argo
```

```text
 Configure MinIO as Artifact Storage

Step 1: Deploy MinIO

Create argo/minio.yaml:

------

apiVersion: v1
kind: Secret
metadata:
  name: minio
  namespace: argo
type: Opaque
stringData:
  accesskey: minio
  secretkey: minio123
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: argo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: minio/minio:latest
          args: ["server", "/data"]
          env:
            - name: MINIO_ROOT_USER
              value: minio
            - name: MINIO_ROOT_PASSWORD
              value: minio123
          ports:
            - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: argo
spec:
  selector:
    app: minio
  ports:
    - port: 9000
      targetPort: 9000

------


Apply MinIO:

kubectl apply -f argo/minio.yaml

Check MinIO pod and service:

kubectl get pods -n argo | findstr minio
kubectl get svc -n argo | findstr minio

Expected:

minio-xxxxx   1/1 Running
minio         ClusterIP   9000/TCP


Step 2: Configure Argo Workflow Controller Artifact Repository

Edit workflow controller configmap:

kubectl edit configmap workflow-controller-configmap -n argo

Add this configuration:

data:
  artifactRepository: |
    s3:
      bucket: argo-artifacts
      endpoint: minio.argo:9000
      insecure: true
      accessKeySecret:
        name: minio
        key: accesskey
      secretKeySecret:
        name: minio
        key: secretkey

Save and exit.

Restart workflow controller:

kubectl rollout restart deployment workflow-controller -n argo
kubectl rollout status deployment workflow-controller -n argo


Verify config:

kubectl get configmap workflow-controller-configmap -n argo -o yaml


Step 3: Create MinIO bucket

```bash
kubectl run minio-client -n argo --rm -it --image=alpine -- sh
```

Inside shell: Run this commands step by step

```sh
apk add --no-cache wget
wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/local/bin/mc
chmod +x /usr/local/bin/mc
mc alias set local http://minio.argo:9000 minio minio123
mc mb --ignore-existing local/argo-artifacts
exit
```

Expected output after cmd - mc ls local:
argo-artifacts/

---


Important Note:

In the current MinIO setup, no PersistentVolume is attached. Because of this, if the MinIO pod or Docker Desktop Kubernetes restarts, the argo-artifacts bucket may be lost.

If the APK workflow fails with:

failed to put file: The specified bucket does not exist

then recreate the bucket:

kubectl run minio-client -n argo --rm -it --image=alpine -- sh

Inside shell:

apk add --no-cache wget
wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/local/bin/mc
chmod +x /usr/local/bin/mc
mc alias set local http://minio.argo:9000 minio minio123
mc mb --ignore-existing local/argo-artifacts
exit

-----

# 16. APK WorkflowTemplate

Create `argo/apk-workflow-template.yaml`:

**Note** : inside this code use your actual Git repo url that you want to use

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: ionic-apk-builder
  namespace: argo
spec:
  serviceAccountName: workflow
  entrypoint: build-apk

  templates:
    - name: build-apk
      container:
        image: harshparmar009/ionic-android-builder:v1
        imagePullPolicy: Always
        command: ["/bin/bash", "-c"]
        args:
          - |
            set -e

            echo "1. Clone Ionic project"
            git clone https://github.com/harshparmar009/test-photo-gallery-react-ionic.git /workspace/app

            cd /workspace/app

            echo "2. Install dependencies"
            npm ci

            echo "3. Build Ionic web app"
            npm run build

            echo "4. Sync Capacitor Android"
            npx cap sync android

            echo "5. Build Android APK"
            cd android
            chmod +x gradlew
            ./gradlew assembleDebug --no-daemon --stacktrace

            echo "6. Copy APK"
            mkdir -p /tmp/apk
            cp app/build/outputs/apk/debug/app-debug.apk /tmp/apk/ionic-app-debug.apk

            echo "APK build completed"
      outputs:
        artifacts:
          - name: ionic-apk
            path: /tmp/apk/ionic-app-debug.apk
```

Apply:

```bash
kubectl apply -f argo/apk-workflow-template.yaml
```

Create `argo/run-apk-workflow.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: ionic-apk-
  namespace: argo
spec:
  workflowTemplateRef:
    name: ionic-apk-builder
```

Run manually:

```bash
kubectl create -f argo/run-apk-workflow.yaml
```

Watch:

```bash
kubectl get workflows -n argo -w
```

Logs:

```bash
kubectl logs -n argo <workflow-name> -f
```

Expected:

```text
BUILD SUCCESSFUL
APK build completed
```

---

# 17. Install Argo Events

Create namespace:

```bash
kubectl create namespace argo-events
```

Install:

```bash
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml

```

Check:

```bash
kubectl get pods -n argo-events -w
```

Expected:

```text
controller-manager   Running
```

# 18. Configure Argo Events Resources

## Step 1: Create GitHub Webhook Secret

Create a Kubernetes secret that stores the GitHub webhook secret.

```bash
kubectl create secret generic github-webhook-secret \
  --from-literal=secret=harsh12345 \
  -n argo
```

Verify:

```bash
kubectl get secrets -n argo
```

Expected:

```text
github-webhook-secret
```

---

## Step 2: Create EventBus

Create `argo/eventbus.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default
  namespace: argo
spec:
  nats:
    native:
      replicas: 1
```

Apply:

```bash
kubectl apply -f argo/eventbus.yaml
```

Verify:

```bash
kubectl get eventbus -n argo
```

Expected:

```text
NAME
default
```

---

## Step 3: Create EventSource

Create `argo/eventsource.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: github-eventsource
  namespace: argo
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000

  webhook:
    github-push:
      endpoint: /github
      method: POST
      port: "12000"
```

Apply:

```bash
kubectl apply -f argo/eventsource.yaml
```

Verify:

```bash
kubectl get eventsource -n argo
kubectl get svc -n argo
```

Expected:

```text
github-eventsource

github-eventsource-eventsource-svc
```

---

## Step 4: Configure Sensor RBAC

Create `argo/sensor-rbac.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sensor-sa
  namespace: argo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sensor-role
  namespace: argo
rules:
- apiGroups: ["argoproj.io"]
  resources:
    - workflows
    - workflowtemplates
  verbs:
    - get
    - list
    - watch
    - create
    - update
    - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sensor-role-binding
  namespace: argo
subjects:
- kind: ServiceAccount
  name: sensor-sa
  namespace: argo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: sensor-role
```

Apply:

```bash
kubectl apply -f argo/sensor-rbac.yaml
```

Verify:

```bash
kubectl get sa -n argo
kubectl get role -n argo
kubectl get rolebinding -n argo
```

---

## Step 5: Create Sensor

Create `argo/sensor.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: github-apk-sensor
  namespace: argo

spec:

  template:
    serviceAccountName: sensor-sa

  dependencies:

    - name: github-push
      eventSourceName: github-eventsource
      eventName: github-push

      filters:
        data:

          - path: body.repository.full_name
            type: string
            value:
              - harshparmar009/test-photo-gallery-react-ionic

          - path: body.ref
            type: string
            value:
              - refs/heads/main

  triggers:

    - template:
        name: trigger-apk-workflow

        argoWorkflow:
          operation: submit

          source:
            resource:

              apiVersion: argoproj.io/v1alpha1
              kind: Workflow

              metadata:
                generateName: ionic-apk-auto-
                namespace: argo

              spec:
                workflowTemplateRef:
                  name: ionic-apk-builder
```

Apply:

```bash
kubectl apply -f argo/sensor.yaml
```

---

## Step 6: Verify Installation

Verify all Argo Events resources.

```bash
kubectl get eventbus,eventsource,sensor -n argo
```

Expected:

```text
NAME                           AGE
eventbus/default

NAME
eventsource/github-eventsource

NAME
sensor/github-apk-sensor
```

Check Pods:

```bash
kubectl get pods -n argo
```

Expected:

```text
argo-server                                  Running
workflow-controller                          Running
eventbus-default-stan-0                      Running
github-eventsource-eventsource-xxxxx         Running
github-apk-sensor-sensor-xxxxx               Running
minio                                        Running
```

---

# 19. Expose Webhook using ngrok

## Step 1 Verify Sensor Configuration

Verify that the Sensor is using the correct repository and branch filters.

```bash
kubectl get sensor github-apk-sensor -n argo -o yaml
```

Verify:

```yaml
body.repository.full_name:
  harshparmar009/test-photo-gallery-react-ionic

body.ref:
  refs/heads/main
```

---

## Step 8: Test Event Trigger

Expose the EventSource Service:

```bash
kubectl port-forward svc/github-eventsource-eventsource-svc -n argo 12000:12000
```

Start ngrok:

```bash
ngrok http 12000
```

copy this ngrok url it should  - https://<your-ngrok-url>.ngrok-free.dev
add - < /github > at the end of this URL

````


# 20. Configure GitHub Webhook

Go to:

```text
GitHub Repository
→ Settings
→ Webhooks
→ Add webhook
```

Set:

```text
Payload URL: https://xxxxx.ngrok-free.dev/github
Content type: application/json
Secret: harsh12345
Events: Just the push event
```

Save.

Test with:

```text
Recent Deliveries → Redeliver
```

or push a commit.

---

# 21. Automatic APK Build Test

Make visible code change.

Example:

```tsx
<h2>APK Auto Build Demo v2</h2>
```

Push:

```bash
git add .
git commit -m "Test automatic APK build"
git push
```

Watch workflow:

```bash
kubectl get workflows -n argo -w
```

Expected:

```text
ionic-apk-auto-xxxxx   Running
```

Logs:

```bash
kubectl logs -n argo ionic-apk-auto-xxxxx -f
```

## if you want to **delete** wokrflow 
cmd - kubectl delete workflow <workflow-pod-name> -n argo

Expected:

```text
1. Clone Ionic project
2. Install dependencies
3. Build Ionic web app
4. Sync Capacitor Android
5. Build Android APK
BUILD SUCCESSFUL
APK build completed
```

Final status:

```bash
kubectl get workflows -n argo
```

Expected:

```text
ionic-apk-auto-xxxxx   Succeeded
```

---

# 22. Argo Workflows UI

Port forward:

```bash
kubectl port-forward svc/argo-server -n argo 2746:2746
```

Open:

```text
https://localhost:2746
```

If login token fails, set server mode:

```bash
kubectl edit deployment argo-server -n argo
```

Change:

```yaml
args:
- server
```

to:

```yaml
args:
- server
- --auth-mode=server
```

Restart:

```bash
kubectl rollout status deployment argo-server -n argo
```

Open in incognito:

```text
https://localhost:2746
```

---

# 23. Download APK

In Argo UI:

```text
Workflows
→ argo namespace
→ ionic-apk-auto-xxxxx
→ Click workflow node
→ Outputs / Artifacts
→ Download ionic-apk
```

Downloaded file:

```text
ionic-apk.tgz
```

Extract using 7-Zip or WinRAR.

Inside:

```text
ionic-app-debug.apk
```

Install APK on Android phone.

---


# 26. Final Explanation

This project implements a complete CI/CD setup for an Ionic Capacitor app.

The web app is deployed using Docker, Kubernetes, and Argo CD.

The Android APK is generated automatically using Argo Events and Argo Workflows.

Whenever code is pushed to GitHub, GitHub sends a webhook event. Argo Events receives the event, the Sensor triggers an Argo Workflow, and the workflow builds the APK inside Kubernetes. The APK is then stored in MinIO and can be downloaded from the Argo Workflows UI.

Final result:

```text
Code Push
↓
GitHub Webhook
↓
Argo Events
↓
Argo Workflow
↓
APK Build
↓
MinIO Artifact
↓
APK Download
```



**************************************************************************************************


**Important**

Order of Starting the CI/CD Pipeline starting from new


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


3. Port-forward the EventSource

Open another terminal:

kubectl port-forward svc/github-eventsource-eventsource-svc -n argo 12000:12000

Keep this terminal open.


4. Start ngrok

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
cmd - kubectl logs -n argo <your-ionic-apk-workflow> -f





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


