# gcp-cicd-gke

## Automating CI/CD Pipelines using Google Cloud Build and GKE

This repository contains a step-by-step guide to setting up an automated CI/CD pipeline using **Google Cloud Build** and **Google Kubernetes Engine (GKE)**.

---

## **1. Prerequisites**

Before starting, ensure you have the following:
- A **Google Cloud Platform (GCP) Account**.
- Installed **gcloud CLI**.
- Enabled necessary APIs:
  ```bash
  gcloud services enable container.googleapis.com cloudbuild.googleapis.com
  ```
- A Kubernetes cluster on GKE.
- A GitHub repository with your application code.

---

## **2. Setting Up the Environment**

1. Authenticate Google Cloud CLI:
   ```bash
   gcloud auth login
   ```
2. Set the active project:
   ```bash
   gcloud config set project [PROJECT_ID]
   ```
3. Create and configure a GKE cluster:
   ```bash
   gcloud container clusters create [CLUSTER_NAME] --zone [ZONE]
   gcloud container clusters get-credentials [CLUSTER_NAME] --zone [ZONE]
   ```

---

## **3. Application and Docker Setup**

### Create a Sample Node.js Application

- `app.js`:
  ```javascript
  const express = require('express');
  const app = express();
  const PORT = process.env.PORT || 8080;
  app.get('/', (req, res) => {
      res.send('Hello, CI/CD with GKE!');
  });
  app.listen(PORT, () => console.log(`App running on port ${PORT}`));
  ```

- `package.json`:
  ```json
  {
    "name": "gke-cicd-app",
    "version": "1.0.0",
    "main": "app.js",
    "dependencies": {
      "express": "^4.18.2"
    }
  }
  ```

### Create a Dockerfile

- `Dockerfile`:
  ```dockerfile
  FROM node:18-slim
  WORKDIR /usr/src/app
  COPY package*.json ./
  RUN npm install
  COPY . .
  EXPOSE 8080
  CMD ["node", "app.js"]
  ```

### Build and Run Locally

```bash
docker build -t gke-cicd-app .
docker run -p 8080:8080 gke-cicd-app
```

---

## **4. Push to Google Container Registry (GCR)**

1. Tag the Docker image:
   ```bash
   docker tag gke-cicd-app gcr.io/[PROJECT_ID]/gke-cicd-app
   ```
2. Push the image to GCR:
   ```bash
   docker push gcr.io/[PROJECT_ID]/gke-cicd-app
   ```

---

## **5. Deploy to GKE**

### Create Kubernetes Deployment & Service YAML

- `deployment.yaml`:
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: gke-cicd-app
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: gke-cicd-app
    template:
      metadata:
        labels:
          app: gke-cicd-app
      spec:
        containers:
          - name: gke-cicd-app
            image: gcr.io/[PROJECT_ID]/gke-cicd-app
            ports:
              - containerPort: 8080
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: gke-cicd-service
  spec:
    type: LoadBalancer
    selector:
      app: gke-cicd-app
    ports:
      - protocol: TCP
        port: 80
        targetPort: 8080
  ```

### Deploy the App

```bash
kubectl apply -f deployment.yaml
```

### Verify Deployment

```bash
kubectl get pods
kubectl get service gke-cicd-service
```

Access the application using the external IP of the service.

---

## **6. Automate CI/CD with Google Cloud Build**

### Create Cloud Build Configuration

- `cloudbuild.yaml`:
  ```yaml
  steps:
    - name: 'gcr.io/cloud-builders/docker'
      args: ['build', '-t', 'gcr.io/$PROJECT_ID/gke-cicd-app', '.']
    - name: 'gcr.io/cloud-builders/docker'
      args: ['push', 'gcr.io/$PROJECT_ID/gke-cicd-app']
    - name: 'gcr.io/cloud-builders/kubectl'
      args:
        - 'apply'
        - '-f'
        - 'deployment.yaml'
    - name: 'gcr.io/cloud-builders/kubectl'
      args:
        - 'set'
        - 'image'
        - 'deployment/gke-cicd-app'
        - 'gke-cicd-app=gcr.io/$PROJECT_ID/gke-cicd-app:$BUILD_ID'
  ```

### Set Up Cloud Build Trigger

```bash
gcloud builds triggers create github \
  --repo-name=[REPO_NAME] \
  --repo-owner=[OWNER] \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml
```

### Push Changes to GitHub

```bash
git add .
git commit -m "Add CI/CD pipeline"
git push origin main
```

### Cloud Build Process:
- Builds the Docker image.
- Pushes it to GCR.
- Deploys or updates the Kubernetes deployment.

---

## **7. Verify the Pipeline**

Check build logs:
```bash
gcloud builds list
gcloud builds log [BUILD_ID]
```

Confirm the deployment:
```bash
kubectl get pods
kubectl describe deployment gke-cicd-app
```

---

## **8. Clean Up**

To remove the created resources:

- Delete the GKE cluster:
  ```bash
  gcloud container clusters delete [CLUSTER_NAME] --zone [ZONE]
  ```
- Delete the Docker image from GCR:
  ```bash
  gcloud container images delete gcr.io/[PROJECT_ID]/gke-cicd-app
  ```

---

