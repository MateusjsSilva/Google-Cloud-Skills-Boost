# Load Balancing Implementation on Compute Engine

## Challenge Scenario
You have just started working as a junior cloud engineer at Jooli Inc. Your role involves managing the company's infrastructure, including provisioning resources for projects.

Supervisors expect you to have the necessary skills and knowledge, so no step-by-step guide is provided.

### Jooli Inc. Guidelines:
- Create all resources in the default region or zone unless instructed otherwise. The default region is `REGION`, and the default zone is `ZONE`.
- Use the naming format `team-resource`, e.g., `nucleus-webserver1`.
- Optimize resource usage to avoid excessive consumption, which could lead to project termination.
- Use `e2-micro` for small Linux VMs and `e2-medium` for Windows or other applications like Kubernetes nodes.

---

## Task 1: Create a "Jumphost" Instance
This instance will be used for project maintenance.

### Requirements:
- Name the instance `INSTANCE`.
- Create it in the zone `ZONE`.
- Use machine type `e2-micro`.
- Use the default image type (Debian Linux).

### Commands:
```bash
export INSTANCE=
export ZONE=
export REGION="${ZONE%-*}"

# Create the jumphost instance
gcloud compute instances create $INSTANCE \
    --zone=$ZONE \
    --machine-type=e2-micro
```

---

## Task 2: Configure an HTTP Load Balancer
You need to set up an HTTP load balancer with a managed instance group of two Nginx web servers.

### Startup Script for Web Servers:
```bash
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```

### Steps:
1. **Create an Instance Template**
```bash
gcloud compute instance-templates create web-server-template \
        --metadata-from-file startup-script=startup.sh \
        --machine-type e2-medium \
        --region $REGION
```

2. **Create a Managed Instance Group**
```bash
gcloud compute instance-groups managed create web-server-group \
        --base-instance-name web-server \
        --size 2 \
        --template web-server-template \
        --region $REGION
```

3. **Create a Firewall Rule**
```bash
gcloud compute firewall-rules create FIREWALL \
        --allow tcp:80 \
        --network default
```

4. **Create a Health Check**
```bash
gcloud compute http-health-checks create http-basic-check
```

5. **Set Named Ports for Instance Group**
```bash
gcloud compute instance-groups managed \
        set-named-ports web-server-group \
        --named-ports http:80 \
        --region $REGION
```

6. **Create a Backend Service**
```bash
gcloud compute backend-services create web-server-backend \
        --protocol HTTP \
        --http-health-checks http-basic-check \
        --global
```

7. **Add Instance Group to Backend Service**
```bash
gcloud compute backend-services add-backend web-server-backend \
        --instance-group web-server-group \
        --instance-group-region $REGION \
        --global
```

8. **Create a URL Map**
```bash
gcloud compute url-maps create web-server-map \
        --default-service web-server-backend
```

9. **Create an HTTP Proxy**
```bash
gcloud compute target-http-proxies create http-lb-proxy \
        --url-map web-server-map
```

10. **Create a Forwarding Rule**
```bash
gcloud compute forwarding-rules create http-content-rule \
      --global \
      --target-http-proxy http-lb-proxy \
      --ports 80
```

11. **List Forwarding Rules**
```bash
gcloud compute forwarding-rules list
```

### Note:
Wait **5 to 7 minutes** to receive the activity score.

---
© Mateus Silva | 2025
