# DevOps
# Trivy Server and DefectDojo Integration

This document explains how to deploy Trivy Server and DefectDojo, and how to integrate them so that Trivy scan results can be automatically imported into DefectDojo.

It includes installation steps, configuration details, and sample CI/CD usage.

## 1. Trivy Overview

Trivy is a vulnerability scanner for container images, file systems, and configurations.

In this setup:

- Trivy runs in server mode on a Bamboo Agent machine (with docker compose).
- A Bamboo agent runs Trivy as a Docker container to scan images and then remove it (with docker run and remove it).
- The scan results can be sent to DefectDojo (on another machine) for vulnerability management and reporting.

## 2. Install Trivy in Server Mode

### 2.1 Docker Compose Deployment

Create a `docker-compose.yml` file:

```yaml
version: "3.3"
services:
  trivy-server:
    image: docker.lib2.tiddev.com/aquasec/trivy:0.69.1
    container_name: trivy-server
    restart: unless-stopped
    volumes:
      - trivy-cache:/root/.cache/trivy
      - ./trivydb/db:/root/.cache/trivy/db
      - ./java-db:/root/.cache/trivy/java-db
    extra_hosts:
      - "ghcr.io:140.82.121.33"
      - "github.com:140.82.121.3"
      - "host.docker.internal:host-gateway"
    environment:
      - TRIVY_DEBUG=true
      - NO_PROXY=localhost,127.0.0.1,::1,*.internal.company.com,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
    command: server --listen 0.0.0.0:4954 --skip-db-update --timeout 30m --debug
    network_mode: host

volumes:
  trivy-cache:
```

### 2.2 Download Trivy Databases (Offline Mode)

If your environment does not allow Trivy to download databases directly, pull them manually:

```bash
oras pull ghcr.io/aquasecurity/trivy-java-db:1 -o ~/trivy-java-db/db
```

## 3. Run Trivy Scans Using a Container

Example: scanning the latest nginx image:

```bash
docker run --rm --network host \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/tplan/trivy/java-db:/root/.cache/trivy/java-db \
  docker.lib2.tiddev.com/aquasec/trivy:0.69.1 \
  image \
  --server http://localhost:4954 \
  --skip-java-db-update \
  --scanners vuln \
  --severity CRITICAL,HIGH \
  --no-progress \
  docker.lib2.tiddev.com/nginx:latest
```

## 4. Install DefectDojo on another VM

### 4.1 Clone Repository

```bash
https://github.com/DefectDojo/django-DefectDojo
```

### 4.2 Start Services (requires internet)

```bash
docker-compose up -d
```

Default Web URL:

```
http://<defectdojo-server-IP>:8080
```

### 4.3 Retrieve Admin Password

```bash
docker logs --tail 100 -f "initializer container" | grep password
```

## 5. Prepare DefectDojo for Receiving Trivy Reports

To import scans, you need:

- API Token
- Product Type
- Product
- Engagement

### 5.1 Generate API Token

Web UI → Admin → API v2 → Generate Token

### 5.2 Get Product Types

```bash
curl -X GET "http://192.168.101.144:8080/api/v2/product_types/" \
  -H "Authorization: Token 9e593f563d1c1d25de5c9c7fd3a4d88d7a47d574"
```

Example: using product type ID = 1

### 5.3 Create a Product

```bash
curl -X POST "http://192.168.101.144:8080/api/v2/products/" \
  -H "Authorization: Token 9e593f563d1c1d25de5c9c7fd3a4d88d7a47d574" \
  -H "Content-Type: application/json" \
  -d '{
   "name": "Container Security",
   "description": "Trivy scans for container images",
   "prod_type": 1
  }'
```

### 5.4 Create Engagement

```bash
curl -X POST "http://192.168.101.144:8080/api/v2/engagements/" \
  -H "Authorization: Token 9e593f563d1c1d25de5c9c7fd3a4d88d7a47d574" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"Trivy CI Pipeline\",\n    \"product\": 1,\n    \"engagement_type\": \"CI/CD\",\n    \"status\": \"In Progress\",\n    \"target_start\": \"2026-04-22\",\n    \"target_end\": \"2027-04-22\"}
"
```

Your DefectDojo instance is now ready.

## 6. Exporting Trivy Reports to DefectDojo

### 6.1 Generate Trivy Report (JSON)

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/tplan/trivy/java-db:/root/.cache/trivy/java-db \
  -v $(pwd):/output \
  docker.lib2.tiddev.com/aquasec/trivy:0.69.1 \
  image \
  --server http://localhost:4954 #(if host mode network)\
  --format json \
  --output /output/trivy-report.json \
  --skip-java-db-update \
  --scanners vuln \
  --severity CRITICAL,HIGH \
  --no-progress \
  docker.lib2.tiddev.com/nginx:latest
```

### 6.2 Upload Report to DefectDojo

```bash
curl -X POST "http://192.168.101.144:8080/api/v2/import-scan/" \
  -H "Authorization: Token 9e593f563d1c1d25de5c9c7fd3a4d88d7a47d574" \
  -F "engagement=2" \
  -F "scan_type=Trivy Scan" \
  -F "file=@trivy-report.json" \
  -F "active=true" \
  -F "verified=true" \
  -F "close_old_findings=true"
```

## 7. CI/CD Pipeline Example (Image Pre-Push Scan)

### 7.1 Generate Report

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/tplan/trivy/java-db:/root/.cache/trivy/java-db \
  -v $(pwd):/output \
  docker.lib2.tiddev.com/aquasec/trivy:0.69.1 \
  image \
  --server http://localhost:4954 #(if host mode network) \
  --format json \
  --output /output/trivy-report.json \
  --skip-java-db-update \
  --scanners vuln \
  --severity CRITICAL,HIGH \
  --no-progress \
  repo.tiddev.com/docker/keyhan/pktb-ekyc:${bamboo.planRepository.branchDisplayName}-${bamboo.buildNumber}
```

### 7.2 Upload CI/CD Scan to DefectDojo

```bash
curl -X POST "http://192.168.101.144:8080/api/v2/import-scan/" \
  -H "Authorization: Token 9e593f563d1c1d25de5c9c7fd3a4d88d7a47d574" \
  -F "engagement=2" \
  -F "scan_type=Trivy Scan" \
  -F "file=@trivy-report.json" \
  -F "active=true" \
  -F "verified=true" \
  -F "close_old_findings=true" \
  -F "build_id=${bamboo.buildNumber}" \
  -F "version=repo.tiddev.com/docker/keyhan/pktb-ekyc:${bamboo_planRepository_branch}-${bamboo_buildNumber}"
```

