# DevOps Assignment – Docker Swarm on AWS

## Overview

In this assignment, I deployed two applications (NestJS and FastAPI) on AWS EC2 using Docker Swarm.

The goal was to create a scalable setup with:

- Replicated services  
- Rolling updates  
- Reverse proxy routing  
- Secrets and configs  
- Named volumes  

I used:

- 1 EC2 instance as Manager  
- 1 EC2 instance as Worker
  
<img width="1918" height="1034" alt="EC2_instances" src="https://github.com/user-attachments/assets/8a46e82c-9605-4b66-b2fc-b4b7ff3997f3" />

---

## 1. Containerization

### NestJS App
- Multi-stage Dockerfile
- Production build only
- `.dockerignore` used to exclude unnecessary files

### FastAPI App
- Python slim image
- Installed dependencies via requirements.txt

Both images were pushed to Docker Hub and pulled on EC2.

---

## 2. Docker Swarm Setup

Initialized swarm:

docker swarm init

Worker joined using join token.

Verified cluster:

docker node ls
<img width="1917" height="153" alt="swarm-cluster" src="https://github.com/user-attachments/assets/bf7bf45e-7cf4-4784-aaf8-efa906f8f738" />

---

## 3. Stack Deployment

Deployed using:

docker stack deploy -c stack.yml multi

Each service configured with:

- replicas: 3  
- restart_policy: on-failure  
- rolling update:
  - parallelism: 1  
  - delay: 10s  

Overlay network used for communication.

Screenshots included:
- Swarm cluster
- Services running
- Replica distribution

<img width="1917" height="186" alt="services-running" src="https://github.com/user-attachments/assets/27b7a1a2-ed49-4843-8921-7051eb4b7f26" />

---

## 4. Secrets, Config and Volume

Created Docker Secret and Config:

docker secret create client_a_db
docker config create client_a_config

Used named volume:

client_a_data

Verified using:

docker secret ls
docker config ls
docker volume ls

<img width="1913" height="195" alt="secrets-configs-volumes" src="https://github.com/user-attachments/assets/5a8a8122-c958-4c9e-9fc2-d13516b1ff70" />

---

## 5. Reverse Proxy (Traefik)

Deployed Traefik as a Swarm service.

Only Traefik exposes port 80.

Routing configured:

- /client-a → NestJS
- /client-b → FastAPI

Tested using:

curl http://<public-ip>/client-a/health
curl http://<public-ip>/client-b/health

Both services responded successfully.

Screenshot included.
<img width="1917" height="153" alt="routing" src="https://github.com/user-attachments/assets/eb2b9d2b-d626-4408-804e-9a1d671d3883" />


---

## 6. Horizontal Scaling

Tested scaling:

docker service scale multi_client-a=6

Containers were distributed across both nodes.

<img width="1913" height="339" alt="scaling" src="https://github.com/user-attachments/assets/e83429fd-1537-41bc-9399-d4c78491947e" />

---

## 7. Rolling Update

Triggered rolling update:

docker service update --force multi_client-a

Containers updated one by one without downtime.

<img width="1913" height="220" alt="rolling-update" src="https://github.com/user-attachments/assets/8bdfaf6d-054c-493a-b7cf-ce483cbcf3c0" />

---
## 8. CI/CD Pipeline Design

To automate build and deployment, a CI/CD pipeline can be implemented using GitHub Actions.

The idea is to automatically build Docker images, push them to a registry, and update the running Swarm services whenever code changes are pushed.

### Pipeline Flow

#### Step 1 – Trigger

- Workflow runs automatically on push to the `main` branch.
- Can also run on pull requests for validation.

#### Step 2 – Build Image

- Docker image is built.
- Image is tagged using commit SHA for version traceability.

Example:

docker build -t shubiie/client-a-node:${{ github.sha }} .

#### Step 3 – Push Image

- Push image to Docker Hub (or AWS ECR in production).

docker push shubiie/client-a-node:${{ github.sha }}

#### Step 4 – Deploy to Swarm

- Pipeline connects to Swarm manager via SSH.
- Update service with new image:

docker service update --image shubiie/client-a-node:${{ github.sha }} multi_client-a

Since Docker Swarm supports rolling updates, deployment happens without downtime.

### Branch-Based Deployment (Bonus)

- dev branch → deploy to development environment  
- staging branch → deploy to staging  
- main branch → deploy to production  

This keeps environments separated and avoids accidental production deployments.

---

## 9. Multi-Client Architecture Strategy

If multiple clients are onboarded regularly (e.g., 20 per month), the architecture should scale while maintaining isolation.

### Shared Cluster Approach

Instead of creating a new cluster per client (which increases cost), a shared Swarm cluster can be used.

Each client can have a separate stack:

- client-a-stack  
- client-b-stack  
- client-c-stack  

This provides logical separation while keeping infrastructure efficient.

### Networking

- Each client can have its own overlay network.
- Reverse proxy (Traefik) routes traffic using domain or path rules.

Example:

- client-a.example.com  
- client-b.example.com  

### Image Versioning

Avoid using `latest` in production.

Use proper version tags:

- client-a-node:1.0.0  
- client-a-node:1.1.0  

This helps in rollback and version tracking.

### Secrets Management

For production:

- Use AWS Secrets Manager or Vault.
- Maintain separate database credentials per client.
- Inject secrets securely into containers.

This improves security and prevents hardcoding credentials.

### Node Labels & Placement

For enterprise clients:

- Use node labels to isolate workloads.
- Apply placement constraints in stack file.

This ensures critical clients can run on dedicated nodes if required.

### Scaling Strategy

- Increase service replicas based on traffic.
- Use AWS Auto Scaling Groups to add worker nodes automatically.
- Swarm schedules containers on newly joined nodes.

### Cost Optimization

- Use shared cluster instead of per-client clusters.
- Use smaller instances for dev/staging.
- Use Spot instances for non-critical workloads.
- Monitor CPU/memory usage and right-size instances accordingly.

---

## 10. Monitoring & Logging

For a production-ready setup, monitoring and centralized logging are essential.

### Monitoring

Use:

- Prometheus for collecting metrics.
- Grafana for dashboards.

Metrics to monitor:

- CPU usage  
- Memory usage  
- Replica count  
- Container restarts  
- Node health  

### Centralized Logging

Option 1:

- Loki + Grafana (lightweight setup)

Option 2:

- ELK Stack (Elasticsearch, Logstash, Kibana)

All container logs should be forwarded to a centralized logging system for easier debugging and analysis.

### Health Checks

Each Docker image should include a health check:

HEALTHCHECK CMD curl -f http://localhost:3000/health || exit 1

If a container becomes unhealthy, Swarm automatically restarts it.
