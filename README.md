# Simple JSON API with CI/CD on Google Cloud

This project deploys a simple JSON API using Python (FastAPI), packaged in Docker, and automatically deployed to Google Cloud Run through a CI/CD pipeline with GitHub Actions.

## Table of Contents
1. [Project Setup](#1-project-setup)
2. [Required IAM Roles](#2-required-iam-roles)
3. [Cost Estimate](#3-cost-estimate)
4. [Design Decisions](#4-design-decisions)

---

## 1. Project Setup

### Prerequisites
* Google Cloud account with a project under the free tier.
* [Google Cloud SDK (gcloud CLI)](https://cloud.google.com/sdk/docs/install) installed.
* GitHub account.
* Docker installed locally (for testing purposes).

### Steps on Google Cloud

1. **Create a GCP Project:**  
   Go to Google Cloud Console to create a new project. After creation, note the Project ID for later use.

2. **Enable Required APIs:**
    ```bash
    gcloud services enable \
        run.googleapis.com \
        artifactregistry.googleapis.com \
        iamcredentials.googleapis.com \
        secretmanager.googleapis.com
    ```

3. **Create an Artifact Registry Repository:**
    ```bash
    gcloud artifacts repositories create my-api-repo \
        --repository-format=docker \
        --location=YOUR_REGION \
        --description="Docker repository for my API" \
        --async
    ```

4. **Create a Service Account for GitHub Actions:**
    ```bash
    gcloud iam service-accounts create github-actions-runner \
        --display-name="GitHub Actions Runner"
    ```

5. **Assign IAM Roles to the Service Account:**
    ```bash
    gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
        --member="serviceAccount:github-actions-runner@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
        --role="roles/run.admin"

    gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
        --member="serviceAccount:github-actions-runner@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
        --role="roles/artifactregistry.writer"

    gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
        --member="serviceAccount:github-actions-runner@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
        --role="roles/iam.serviceAccountUser"
    ```

6. **Create and Download the Service Account Key:**  
   This key will be added to GitHub Secrets.
    ```bash
    gcloud iam service-accounts keys create gcp-credentials.json \
        --iam-account="github-actions-runner@YOUR_PROJECT_ID.iam.gserviceaccount.com"
    ```

### GitHub Setup

1. Fork/clone this repository.
2. Go to `Settings > Secrets and variables > Actions` in your GitHub repository.
3. Create the following secrets:
   * `GCP_PROJECT_ID`: Your Google Cloud project ID (e.g., `my-cicd-project-12345`).
   * `GCP_SA_KEY`: Paste the full contents of the `gcp-credentials.json` file created above.
   * `REGION`: The region where you created Artifact Registry and want to deploy Cloud Run.
   * `REPO_NAME`: The name of the repository you created in Artifact Registry.

Once complete, every push to the `main` branch will trigger the GitHub Actions workflow to build, test, and deploy the application automatically.

---

## 2. Required IAM Roles

The project follows the principle of least privilege. The `github-actions-runner` Service Account is granted the following roles:

* `roles/run.admin` (Cloud Run Admin): Required to deploy new revisions, update services, and route traffic in Cloud Run.
* `roles/artifactregistry.writer` (Artifact Registry Writer): Allows pushing built Docker images to Artifact Registry.
* `roles/iam.serviceAccountUser` (Service Account User): Required for Cloud Run to impersonate this Service Account during deployment, a common security requirement.

---

## 3. Cost Estimate

This project is designed to stay within the **Google Cloud Free Tier** limits.

* **GitHub Actions:** Free for public repositories.
* **Cloud Run:** Free tier provides 2 million requests, 360,000 vCPU-seconds, and 180,000 GiB-seconds per month. "Hello World" usage will be negligible, almost certainly **$0**.
* **Artifact Registry:** Free tier provides 0.5 GB storage per month. The Docker image is very small (<100 MB), so storage cost is **$0**.
* **Cloud Logging:** Free tier includes 50 GiB logs per month. Usage will be minimal.

**Summary:** Expected monthly cost is **$0**, assuming traffic stays within free tier limits.

---

## 4. Design Decisions

### Technology

* **Language:** Python with **FastAPI** for high performance, modern syntax, and lightweight JSON APIs.
* **Docker Multi-stage Build:** The build process uses multiple stages. The first stage installs dependencies and runs tests. The final stage copies only the ready-to-run application into a lightweight base image. This reduces final image size and removes unnecessary build tools in production, improving security.
* **Security Scanning:** **Trivy** is integrated into CI to scan Docker images for known vulnerabilities in OS packages and application libraries before pushing to Artifact Registry.

### Platform and Automation

* **CI/CD - GitHub Actions:** Chosen for tight integration with GitHub repositories. Secrets management, log viewing, and workflow definitions are all in `.github/workflows/deploy.yaml`.
* **Deployment - `gcloud` CLI:** Scripts in GitHub Actions use `gcloud` CLI instead of Terraform. For simple projects with minimal infrastructure changes, `gcloud` is fast, intuitive, and easy to debug.

### Security

* **Non-root User in Docker:** The Docker image runs the application as a non-root user, minimizing risks if the app is compromised.
* **Secrets Management:** All sensitive information, such as Service Account keys and Project ID, is stored in **GitHub Actions Secrets** to prevent exposure in code or CI/CD logs.
