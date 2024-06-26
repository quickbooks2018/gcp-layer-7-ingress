# GCP Layer7 Ingress

- 1. Use Self Signed Certificate
```bash
cd certs
./tls.sh
```

- 2. Create Self Signed Certificate for application, which will be use by GKE ingress
```bash
kubectl create secret tls saqlainmushtaq-com-tls --cert=saqlainmushtaq.com.crt --key=saqlainmushtaq.com.key -n default
k get secrets -n default
```

- 3. helm install
```bash
helm -n default upgrade --install hello --create-namespace -f values.yaml ./ --wait
k get ingress -n default
k describe ingress -n default
```

- 4. See Load Balancing in GCP

- 5. Ggcp Cloud Armor to allow cloudflare ip's and your own ip's
 
- Set GCP default project
```bash
gcloud config set project [PROJECT_ID]
```

### Note: Give 5 Minutues to each network policy to take effect, it is not aws NACL, it takes time to apply

```bash
gcloud compute security-policies create my-cloud-armor-policy --description="Policy to allow Cloudflare and specific IPs"
```

- 6a add rules in policy
```bash
gcloud compute security-policies rules create 1000 \
  --security-policy=my-cloud-armor-policy \
  --src-ip-ranges=173.245.48.0/20,103.21.244.0/22,103.22.200.0/22,103.31.4.0/22,141.101.64.0/18,108.162.192.0/18,190.93.240.0/20,188.114.96.0/20,197.234.240.0/22,198.41.128.0/17 \
  --action="allow" \
  --description="Allow Cloudflare IPv4 ranges (1)"
```
- 6b add rules in policy
```bash
gcloud compute security-policies rules create 1010 \
  --security-policy=my-cloud-armor-policy \
  --src-ip-ranges=162.158.0.0/15,104.16.0.0/12,172.64.0.0/13,131.0.72.0/22 \
  --action="allow" \
  --description="Allow Cloudflare IPv4 ranges (2)"
```

- 6c Allow vpn IP
```bash
gcloud compute security-policies rules create 1020 \
  --security-policy=my-cloud-armor-policy \
  --src-ip-ranges=39.51.215.113/32 \
  --action="allow" \
  --description="Allow traffic from my VPN IP"
```

- 6d Allow Load Balancer IP
```bash
gcloud compute security-policies rules create 1030 \
  --security-policy=my-cloud-armor-policy \
  --src-ip-ranges=34.160.92.28  \
  --action="allow" \
  --description="Allow traffic from Load Balancer IP"
```

- 6d Deny All IP accept IP's (Default Rule can not be deleted to updated below)
```bash
gcloud compute security-policies rules update 2147483647 \
    --security-policy=my-cloud-armor-policy \
    --action="deny-404" \
    --description="Default rule to deny all traffic"
```

- 6e Deny All IPs
```bash
gcloud compute security-policies rules create 999 \
    --security-policy=my-cloud-armor-policy \
    --action="deny-404" \
    --description="Default rule to deny all traffic" \
    --src-ip-ranges="*"  # Use * to apply to all IP addresses
```

7. Associate the security policy with the backend service
```bash
gcloud compute backend-services list

gcloud compute backend-services update k8s1-54998642-default-hello-8080-20522725 \
  --security-policy=my-cloud-armor-policy \
  --global

gcloud compute backend-services update k8s1-54998642-kube-system-default-http-backend-80-ae92c5d3 \
  --security-policy=my-cloud-armor-policy \
  --global
```

8. Verify the security policy is associated with the backend service
```bash
gcloud compute backend-services describe k8s1-54998642-default-hello-8080-20522725 --global

gcloud compute security-policies describe my-cloud-armor-policy
```

8. deny rule '2147483647' is the highest priority rule in the policy
```bash
gcloud compute security-policies rules update 2147483647 \
  --security-policy=my-cloud-armor-policy \
  --action="deny-404" \
  --description="Default deny rule"
```
