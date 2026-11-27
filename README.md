# Kakushin
Kakushin (核心): Japanese for “core” or “heart of the matter.” The central foundation of the MadWorld ecosystem, providing shared services for projects like Meishi and Byakko, including a central authentication server and a centralized logging server. One core, many projects.

## Getting started (VPS)

The VPS chart in `Deployments/VPS` sets up the central authentication server and the backups for the VPS:

- Keycloak with a PostgreSQL database, available at `https://authentication.<domain>`
- Velero, which backs up the files on the stateful pods' volumes (PVCs) every day to OVHcloud Object Storage

### Requirements

- A Kubernetes cluster on the VPS, with `kubectl` and `helm` pointing at it
- Traefik as the ingress controller and cert-manager installed in the cluster
- A domain with a DNS record you can manage
- The [Velero CLI](https://velero.io/docs/main/basic-install/#install-the-cli) (optional, for checking backups)
- An OVHcloud Object Storage bucket and an S3 user with access to it
  (OVH Control Panel → Public Cloud → Object Storage)

### 1. Install Velero

Install the Velero server with the AWS (S3) plugin and the node-agent:

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update
helm upgrade --install velero vmware-tanzu/velero \
  -n velero --create-namespace -f Deployments/Velero/values.yaml
```

Wait until the pods are running:

```bash
kubectl -n velero get pods
```

### 2. Add the secrets and the domain to GitHub

Passwords and keys are not stored in Git. Add them as secrets to the `vps-production` environment in
GitHub (repository **Settings → Environments → vps-production → Environment secrets**):

- `KUBECONFIG`: the kubeconfig of the cluster, base64-encoded (`base64 -i ~/.kube/config`)
- `OVH_S3_ACCESS_KEY_ID`: the access key of your OVH S3 user
- `OVH_S3_SECRET_ACCESS_KEY`: the secret key of your OVH S3 user
- `POSTGRES_PASSWORD`: the password of the Keycloak database
- `KEYCLOAK_ADMIN_PASSWORD`: the password of the Keycloak bootstrap admin (user `admin`)
- `CLUSTER_ISSUER_EMAIL`: the email address Let's Encrypt sends certificate expiry notices to

Choose `POSTGRES_PASSWORD` and `KEYCLOAK_ADMIN_PASSWORD` before the first deploy. PostgreSQL and Keycloak
only read them on the first start, so changing the secrets later does not change the passwords. Change the
admin password in Keycloak itself.

Add the domain as a variable to the same environment (**Environment variables**, not secrets):

- `DOMAIN`: the base domain, for example `mad-world.eu`. Keycloak runs at `authentication.<DOMAIN>`.

Then add a DNS record (A record) for `authentication.<DOMAIN>` that points to the IP address of the VPS.
Let's Encrypt checks this hostname over port 80 before it issues the certificate.

The deploy workflow (`.github/workflows/deploy-production.yml`) passes the secrets and the domain to Helm
in step 4. The chart creates the Kubernetes Secrets from them, such as `velero-ovh-credentials`.

### 3. Check the settings

In `Deployments/VPS/values.yaml`, check:

- `ingress.domain`, `postgres.password`, `keycloak.adminPassword` and `clusterIssuer.email`: local
  defaults only, the pipeline overrides them
- `velero.storageLocation.bucket`: your bucket name
- `velero.storageLocation.region`: your bucket's region (for example `de` or `gra`)
- `velero.schedules[].includedNamespaces`: the namespaces to back up
- `velero.credentials`: leave `existingSecret`, `accessKeyId` and `secretAccessKey` empty (the keys come from the pipeline)

### 4. Install the VPS chart

The pipeline runs:

```bash
helm upgrade --install kakushin Deployments/VPS \
  --values Deployments/VPS/values.yaml \
  --set clusterIssuer.enabled=true \
  --set-string clusterIssuer.email="$CLUSTER_ISSUER_EMAIL" \
  --set-string ingress.domain="$DOMAIN" \
  --set-string postgres.password="$POSTGRES_PASSWORD" \
  --set-string keycloak.adminPassword="$KEYCLOAK_ADMIN_PASSWORD" \
  --set-string velero.credentials.accessKeyId="$OVH_S3_ACCESS_KEY_ID" \
  --set-string velero.credentials.secretAccessKey="$OVH_S3_SECRET_ACCESS_KEY" \
  --wait \
  --timeout 5m
```

### 5. Check that Keycloak works

```bash
# Keycloak and PostgreSQL should be running
kubectl -n kakushin get pods

# The certificate should show "True" under READY
kubectl -n kakushin get certificate
```

Then open `https://authentication.<DOMAIN>` and log in to the admin console with `admin` and
`KEYCLOAK_ADMIN_PASSWORD`.

### 6. Check that backups work

```bash
# The bucket connection should show "Available"
velero backup-location get

# Run a backup now instead of waiting for the schedule
velero backup create test --from-schedule daily --wait

# Check that the pod volumes were backed up
velero backup describe test --details
```

### Restore

```bash
velero backup get
velero restore create --from-backup <backup-name>
```
