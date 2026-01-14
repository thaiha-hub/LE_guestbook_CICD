# Automated Guestbook Deployment with GitHub Actions 
  
**Course:** CI/CD  
**Architecture:** Push-based Deployment

## Project Overview
This project automates the lifecycle of a 3-tier Guestbook application (Frontend, Backend, Database, Redis).

A push-based CI/CD pipeline using Github Actions, where each code push automatically builds Docker images and deploys them to the Openshift cluster.

## The Pipeline Workflow
The workflow is defined in `.github/workflows/pipeline.yml` and consists of two main jobs in the workflow:

### 1. Build (CI)
* **Trigger:** Pushing code to the `main` branch.
* **Action:**
    * Checks out the code.
    * Builds Docker images for both Frontend and Backend.
    * Pushes these images to GitHub Container Registry (GHCR).

### 2. Deploy (CD)
* **Dependency:** Runs only if the Build stage succeeds.
* **Action:**
    * Installs the OpenShift CLI (`oc`) in the GitHub runner.
    * Log in to the OpenShift cluster using a secure Token stored in GitHub Secrets.
    * Apply  the Kubernetes manifests (`oc apply -f k8s/`).
    * Restart the deployments (`oc rollout restart`) to force the pods to pull the new images immediately.

## Tech Stack
* **CI/CD:** GitHub Actions
* **Container Registry:** GHCR (ghcr.io)
* **Orchestrator:** Red Hat OpenShift
* **Application:** Go (Backend), HTML/JS (Frontend), Redis, PostgreSQL.

## Configuration & Secrets
For this pipeline to work, the GitHub repository required the following **Secrets** to be configured (Settings -> Secrets and variables -> Actions):

| Secret Name | Description |
| :--- | :--- |
| `OPENSHIFT_SERVER` | The API URL of the OpenShift Cluster (e.g., `https://api.sandbox...`) |
| `OPENSHIFT_TOKEN` | The Service Account token allowing GitHub to log in to the cluster. |

## Issues & troubleshooting

**Database Persistence (PVC):**
    * **Issue:** Updating the `DB_PASSWORD` Secret did not work; the database kept rejecting the login.
    * **Solution:** I learned that PostgreSQL initializes the data directory only once. I had to delete the Persistent Volume Claim (PVC) to force the database to re-initialize with the new password.

**OpenShift Route & Port Naming:**
    * **Issue:** The application deployed successfully but could not be reached via the public url
    * **Solution:** OpenShift Routes require the Service port to have a specific name.
        * In `service.yaml`, I added `name: http` to the frontend Service port. 
        * In `route.yaml`, I updated `targetPort` to reference that name (`targetPort: http`) instead of the number.

##  Repository Structure
```text
├── .github/workflows
│   └── pipeline.yml       # The CI/CD Pipeline definition
├── backend/           # Source code for Backend
├── frontend/          # Source code for Frontend
├── k8s/               # Kubernetes Manifests
│   ├── route.yaml
│   ├── service.yaml
│   └── ...
└── README.md          # Project Documentation

