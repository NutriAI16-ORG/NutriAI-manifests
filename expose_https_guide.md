# Detailed Steps: Exposing NutriAI (Prod/Dev) and Argo CD over HTTPS with `nutriai.buzz`

This guide outlines the exact, end-to-end steps to generate a single wildcard SSL/TLS certificate for **`nutriai.buzz`**, upload it to your Azure Key Vaults, and configure HTTPS for **Production**, **Development**, and **Argo CD**.

---

## 🔐 Azure Permissions & VM Identity Setup

If you are running the `az keyvault certificate import` command from a self-hosted runner or a VM, the VM's **System-Assigned Managed Identity** (or the Service Principal running your pipeline) must be granted write permissions to the Key Vault.

If your Key Vault uses **Azure RBAC** (Role-Based Access Control):
1. **Role Needed**: **`Key Vault Certificates Officer`** (gives permission to import and manage certificates).
2. **Command to Assign Role**:
   Run this command from an account with Owner/User Access Administrator permissions (replace `<VM_IDENTITY_OBJECT_ID>` with your VM's System-Assigned Managed Identity Object ID):
   ```bash
   # Assign permissions for Production Key Vault
   az role assignment create \
     --role "Key Vault Certificates Officer" \
     --assignee-object-id "<VM_IDENTITY_OBJECT_ID>" \
     --scope "/subscriptions/<YOUR_SUBSCRIPTION_ID>/resourcegroups/nutriai-rg-prod/providers/Microsoft.KeyVault/vaults/nutriai-kv-prod-v3" \
     --assignee-principal-type "ServicePrincipal"

   # Assign permissions for Development Key Vault
   az role assignment create \
     --role "Key Vault Certificates Officer" \
     --assignee-object-id "<VM_IDENTITY_OBJECT_ID>" \
     --scope "/subscriptions/<YOUR_SUBSCRIPTION_ID>/resourcegroups/nutriai-rg-dev/providers/Microsoft.KeyVault/vaults/nutriai-kv-dev-v3" \
     --assignee-principal-type "ServicePrincipal"
   ```

If your Key Vault uses **Access Policies**:
1. **Access Permissions Needed**: `Get`, `List`, `Import` under **Certificate permissions**.
2. **Command to Assign Policy**:
   ```bash
   az keyvault set-policy --name "nutriai-kv-prod-v3" --object-id "<VM_IDENTITY_OBJECT_ID>" --certificate-permissions get list import
   ```

---

## 📋 Target Subdomain Mapping
A single wildcard certificate for `*.nutriai.buzz` and `nutriai.buzz` will cover all three destinations:
* **Production App**: `https://nutriai.buzz`
* **Development App**: `https://dev.nutriai.buzz`
* **Argo CD Dashboard**: `https://argocd.nutriai.buzz`

---

## 🛠️ Phase 1: Install Certbot

Before generating certificates, you must install Certbot. Choose the instructions matching your operating system (Ubuntu is recommended since your VM runs Ubuntu).

### Option A: Linux (Ubuntu) — Recommended
Let's Encrypt officially recommends installing Certbot via `snap` on Ubuntu to ensure you get the latest version.

1. **Update packages and ensure snapd is installed**:
   ```bash
   sudo apt update
   sudo apt install snapd -y
   ```
2. **Remove any pre-existing Certbot installations (to avoid conflicts)**:
   ```bash
   sudo apt remove certbot -y
   ```
3. **Install Certbot using snap**:
   ```bash
   sudo snap install --classic certbot
   ```
4. **Create a symlink to make the command available globally**:
   ```bash
   sudo ln -s /snap/bin/certbot /usr/bin/certbot
   ```
5. **Verify installation**:
   ```bash
   certbot --version
   ```

---

### Option B: macOS
If you are generating the certificate locally on a Mac:
1. Open Terminal and install using Homebrew:
   ```bash
   brew install certbot
   ```

---

### Option C: Windows
If you are running locally on Windows:
1. Download the latest Certbot installer from the [official Certbot Windows Installer](https://dl.eff.org/certbot-beta-installer-win32.exe).
2. Run the installer and follow the wizard.
3. Open a new Command Prompt or PowerShell window as Administrator and verify:
   ```cmd
   certbot --version
   ```

---

## 🔑 Phase 2: Generate the Wildcard Certificate

We will use Let's Encrypt (Certbot) to issue a single wildcard certificate using your email: **`<YOUR_EMAIL_ADDRESS>`**.

### 1. Run the Certbot DNS Challenge
Run this command to request a certificate covering both the root domain and all subdomains:
```bash
sudo certbot certonly --manual --preferred-challenges dns \
  --email "<YOUR_EMAIL_ADDRESS>" \
  --agree-tos \
  --no-eff-email \
  -d "nutriai.buzz" \
  -d "*.nutriai.buzz"
```

### 2. Verify Domain Ownership (DNS TXT Record)
1. Certbot will pause and display a prompt similar to:
   ```text
   Please deploy a DNS TXT record under the name
   _acme-challenge.nutriai.buzz with the following value:
   crT34j...random_token...
   ```
2. Log in to your domain registrar where **`nutriai.buzz`** is registered (e.g., Hostinger, GoDaddy, Namecheap, or Azure DNS).
3. Create a new DNS record with the following details:
   * **Type**: `TXT`
   * **Host/Name**: `_acme-challenge` or `_acme-challenge.nutriai.buzz`
   * **Value**: Copy the random token provided by Certbot.
   * **TTL**: `3600` (or `600` for faster validation).
4. After adding it, wait 1-2 minutes for propagation, then press **Enter** in the Certbot terminal to finalize the validation.

Your certificates will be generated and stored under:
* **Certificate Chain**: `/etc/letsencrypt/live/nutriai.buzz/fullchain.pem`
* **Private Key**: `/etc/letsencrypt/live/nutriai.buzz/privkey.pem`

---

## 📦 Phase 3: Convert to PFX and Upload to Azure Key Vaults

Since you are running both Dev and Prod environments, you will upload the same certificate to both Key Vaults:
* Production Key Vault: **`nutriai-kv-prod-v3`**
* Development Key Vault: **`nutriai-kv-dev-v3`**

### 1. Convert PEM files to PFX format
Run this command to combine the cert and key into a PKCS#12 `.pfx` file (set a memorable export password when prompted):
```bash
sudo openssl pkcs12 -export -out nutriai-wildcard.pfx \
  -inkey /etc/letsencrypt/live/nutriai.buzz/privkey.pem \
  -in /etc/letsencrypt/live/nutriai.buzz/fullchain.pem
```

### 2. Upload to Key Vaults
Use the Azure Portal or Azure CLI to import the `.pfx` file to **both** vaults:

#### For Production (`nutriai-kv-prod-v3`)
```bash
az keyvault certificate import \
  --vault-name "nutriai-kv-prod-v3" \
  --name "nutriai-tls-cert" \
  --file "nutriai-wildcard.pfx" \
  --password "YOUR_EXPORT_PASSWORD"
```

#### For Development (`nutriai-kv-dev-v3`)
```bash
az keyvault certificate import \
  --vault-name "nutriai-kv-dev-v3" \
  --name "nutriai-tls-cert" \
  --file "nutriai-wildcard.pfx" \
  --password "YOUR_EXPORT_PASSWORD"
```

*Note: In both Key Vaults, the certificate must be named **`nutriai-tls-cert`** to match the ingress configurations below.*

---

## 🛡️ Phase 4: Grant Key Vault Access to Application Gateway

Ensure that the Application Gateway's Managed Identity (AGIC) has certificate read access. Run these commands using the User-Assigned Managed Identity's Object ID:

```bash
# Grant access to Prod Key Vault
az role assignment create \
  --role "Key Vault Certificates User" \
  --assignee-object-id <PROD_AGIC_IDENTITY_OBJECT_ID> \
  --scope "/subscriptions/<YOUR_SUBSCRIPTION_ID>/resourcegroups/nutriai-rg-prod/providers/Microsoft.KeyVault/vaults/nutriai-kv-prod-v3" \
  --assignee-principal-type "ServicePrincipal"

# Grant access to Dev Key Vault
az role assignment create \
  --role "Key Vault Certificates User" \
  --assignee-object-id <DEV_AGIC_IDENTITY_OBJECT_ID> \
  --scope "/subscriptions/<YOUR_SUBSCRIPTION_ID>/resourcegroups/nutriai-rg-dev/providers/Microsoft.KeyVault/vaults/nutriai-kv-dev-v3" \
  --assignee-principal-type "ServicePrincipal"
```

---

## 🚀 Phase 5: Expose Production and Dev Applications

### 1. Update Ingress Template
Ensure `ingress.yaml` enables SSL redirection and references the certificate:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nutriai-ingress
  namespace: {{ .Values.namespace }}
  annotations:
    argocd.argoproj.io/sync-wave: "5"
    appgw.ingress.kubernetes.io/request-timeout: "300"
    appgw.ingress.kubernetes.io/connection-draining: "true"
    appgw.ingress.kubernetes.io/connection-draining-timeout: "30"
    # Enable SSL Redirect and Key Vault integration
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: "nutriai-tls-cert"
spec:
  ingressClassName: azure-application-gateway
  rules:
    {{- if .Values.ingress.host }}
    - host: {{ .Values.ingress.host }}
      http:
    {{- else }}
    - http:
    {{- end }}
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 8000
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 8000
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

### 2. Configure Custom Hosts in Values Files
Modify the hosts in both environment values files to match your domain and subdomains:

* **Production Configuration (`values-prod.yaml`):**
  ```yaml
  ingress:
    host: "nutriai.buzz"
  ```

* **Development Configuration (`values-dev.yaml`):**
  ```yaml
  ingress:
    host: "dev.nutriai.buzz"
  ```

Commit and sync these modifications using Argo CD.

---

## 🐙 Phase 6: Expose Argo CD over HTTPS

### 1. Configure Argo CD Server for HTTP Backend
Ensure Argo CD runs in insecure mode since the Application Gateway will handle SSL/TLS termination:
```bash
kubectl patch deployment argocd-server -n argocd --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--insecure"}]'
```

### 2. Deploy the Ingress Resource for Argo CD
Create `argocd-ingress.yaml` to route `argocd.nutriai.buzz` securely:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: azure-application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/backend-protocol: "Http"
    # Point to the wildcard certificate in your Key Vault
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: "nutriai-tls-cert"
spec:
  rules:
    - host: "argocd.nutriai.buzz"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

Apply this ingress resource:
```bash
kubectl apply -f argocd-ingress.yaml
```

---

## 🔍 Phase 7: DNS Verification & Routing

Once the configurations are applied:
1. Log in to your domain registrar's DNS settings panel.
2. Create **three A records** pointing to your Application Gateway's Public IP address (`20.110.121.69`):
   * **Record 1 (Root Prod)**: Host: `@` (or leave empty) -> Points to: `20.110.121.69`
   * **Record 2 (Dev Subdomain)**: Host: `dev` -> Points to: `20.110.121.69`
   * **Record 3 (Argo CD Subdomain)**: Host: `argocd` -> Points to: `20.110.121.69`

Once DNS propagates (usually within a few minutes), navigate to:
* `https://nutriai.buzz` (Prod App)
* `https://dev.nutriai.buzz` (Dev App)
* `https://argocd.nutriai.buzz` (Argo CD Dashboard)
