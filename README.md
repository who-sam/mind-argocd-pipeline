HelloApp – CI/CD on EKS using Jenkins & ArgoCD (Short README)

Overview

A simple CI/CD pipeline that builds a Docker image using Jenkins and deploys it automatically to an Amazon EKS cluster using ArgoCD (GitOps).

⸻

Flow
	1.	Jenkins (CI)
	•	Builds Docker image from Dockerfile
	•	Pushes image to Docker Hub
	2.	ArgoCD (CD)
	•	Watches a GitHub repo containing Helm chart
	•	Syncs changes automatically to EKS
	•	Deploys the updated app
	3.	EKS (Kubernetes)
	•	Runs the application
	•	Exposes it using a LoadBalancer Service

Repo Structure
app-cd/
    deployment.yaml

How Deployment Works
	•	Update code → Jenkins builds & pushes image
	•	Update Helm chart repo → ArgoCD auto-syncs
	•	EKS rolls out the new version automatically

⸻

Access App:
kubectl get svc

Copy the EXTERNAL-IP and open it in browser.

⸻

Used Technologies:
	•	Terraform
	•	Jenkins
	•	Docker
	•	ArgoCD
	•	Amazon EKS


    