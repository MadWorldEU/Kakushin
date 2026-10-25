# Kakushin
Kakushin (核心): Japanese for “core” or “heart of the matter.” The central foundation of the MadWorld ecosystem, providing shared services for projects like Meishi and Byakko, including a central authentication server and a centralized logging server. One core, many projects.

## Getting started (VPS)

The VPS chart in `Deployments/VPS` sets up the backups for the VPS. Velero backs up the files on the
stateful pods' volumes (PVCs) every day to OVHcloud Object Storage.

### Requirements

- A Kubernetes cluster on the VPS, with `kubectl` and `helm` pointing at it
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

### 2. Add the OVH credentials to GitHub

The OVH S3 keys are not stored in Git. Add them as secrets to the `vps-production` environment in
GitHub (repository **Settings → Environments → vps-production → Environment secrets**):

- `OVH_S3_ACCESS_KEY_ID`: the access key of your OVH S3 user
- `OVH_S3_SECRET_ACCESS_KEY`: the secret key of your OVH S3 user

The deploy workflow (`.github/workflows/deploy-production.yml`) passes them to Helm in step 4, and the
chart creates the `velero-ovh-credentials` Secret from them.

### 3. Check the settings

In `Deployments/VPS/values.yaml`, check:

- `velero.storageLocation.bucket`: your bucket name
- `velero.storageLocation.region`: your bucket's region (for example `de` or `gra`)
- `velero.schedules[].includedNamespaces`: the namespaces to back up
- `velero.credentials`: leave `existingSecret`, `accessKeyId` and `secretAccessKey` empty (the keys come from the pipeline)

### 4. Install the VPS chart

The pipeline runs:

```bash
helm upgrade --install kakushin Deployments/VPS \
  --values Deployments/VPS/values.yaml \
  --set-string velero.credentials.accessKeyId="$OVH_S3_ACCESS_KEY_ID" \
  --set-string velero.credentials.secretAccessKey="$OVH_S3_SECRET_ACCESS_KEY" \
  --wait \
  --timeout 5m
```

### 5. Check that backups work

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
