# TLS Certificate Renewal and Kubernetes Secret Update Guide

## Prerequisites

* Access to Linux jump host
* kubectl configured and working
* Docker installed
* Access to DNS provider for TXT record updates
* Replace all example values with your actual domain, namespace, and secret names

## Example Values

| Item       | Example                 |
| ---------- | ----------------------- |
| Domain     | example-uat.company.com |
| Namespace  | app-uat                 |
| TLS Secret | app-tls                 |

---

## Step 1: Check Current Certificate Expiry

```bash
echo | openssl s_client \
-servername example-uat.company.com \
-connect example-uat.company.com:443 2>/dev/null \
| openssl x509 -noout -issuer -subject -dates
```

Quick expiry check:

```bash
echo | openssl s_client \
-servername example-uat.company.com \
-connect example-uat.company.com:443 2>/dev/null \
| openssl x509 -noout -enddate
```

---

## Step 2: Generate / Renew Let's Encrypt Certificate

```bash
sudo docker run -it --rm --name certbot \
-v "/etc/letsencrypt:/etc/letsencrypt" \
-v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
certbot/certbot certonly \
-d example-uat.company.com \
--manual \
--preferred-challenges dns \
--email your.name@company.com \
--agree-tos \
--no-eff-email
```

Certbot will provide a TXT record value.

---

## Step 3: Create DNS TXT Record

Create:

```text
_acme-challenge.example-uat.company.com
```

Value:

```text
<value provided by certbot>
```

---

## Step 4: Validate DNS Propagation

```bash
dig TXT _acme-challenge.example-uat.company.com @8.8.8.8
```

or

```bash
nslookup -type=TXT _acme-challenge.example-uat.company.com
```

Verify the TXT value matches the value shown by Certbot.

---

## Step 5: Verify Certificate Files

```bash
sudo ls -l /etc/letsencrypt/live/example-uat.company.com/
```

Expected files:

```text
fullchain.pem
privkey.pem
```

---

## Step 6: Backup Existing Kubernetes Secret

```bash
kubectl get secret app-tls -n app-uat -o yaml > app-tls-backup.yaml
```

---

## Step 7: Create Local Copies (Avoid Permission Issues)

```bash
sudo cp /etc/letsencrypt/live/example-uat.company.com/fullchain.pem .
sudo cp /etc/letsencrypt/live/example-uat.company.com/privkey.pem .
sudo chown $USER:$USER fullchain.pem privkey.pem
```

---

## Step 8: Update TLS Secret

If secret already exists:

```bash
kubectl delete secret app-tls -n app-uat
```

Create new secret:

```bash
kubectl create secret tls app-tls \
-n app-uat \
--cert=fullchain.pem \
--key=privkey.pem
```

---

## Step 9: Verify Secret Certificate Expiry

```bash
kubectl get secret app-tls -n app-uat \
-o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

---

## Step 10: Verify Public Endpoint Uses New Certificate

```bash
echo | openssl s_client \
-servername example-uat.company.com \
-connect example-uat.company.com:443 2>/dev/null \
| openssl x509 -noout -dates
```

Confirm the Not After date matches the newly issued certificate.

---

# Troubleshooting

## Permission Denied Reading Certificate Files

```bash
sudo cp /etc/letsencrypt/live/example-uat.company.com/fullchain.pem .
sudo cp /etc/letsencrypt/live/example-uat.company.com/privkey.pem .
sudo chown $USER:$USER fullchain.pem privkey.pem
```

## kubectl Connection Refused When Using sudo

Do not use:

```bash
sudo kubectl
```

Verify access:

```bash
kubectl get nodes
```

## Verify TLS Secret Name Used by Ingress

```bash
kubectl get ingress -A
kubectl describe ingress <ingress-name> -n <namespace>
```

Confirm the ingress references the expected TLS secret.
