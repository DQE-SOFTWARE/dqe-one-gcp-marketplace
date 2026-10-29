# DQE-One on Google Kubernetes Engine

DQE-One is a data quality platform by [DQE Software](https://www.dqe-software.com): contact data validation, standardization and deduplication.

This repository contains the Kubernetes manifests used by the Google Cloud Marketplace listing of DQE-One, and the instructions to install it from the command line.

- [Installation](#installation)
  - [Quick install with Google Cloud Marketplace](#quick-install-with-google-cloud-marketplace)
  - [Command-line instructions](#command-line-instructions)
- [Access DQE-One](#access-dqe-one)
- [Expose DQE-One on your own domain](#expose-dqe-one-on-your-own-domain)
- [Update](#update)
- [Uninstall](#uninstall)

## Architecture

The application is made of the following components, all deployed in a single namespace:

| Component | Kubernetes resource | Role |
|---|---|---|
| DQE-One | `Deployment` + `Service` (port 8001) | Django web application |
| PostgreSQL | `StatefulSet` + headless `Service` + `PersistentVolumeClaim` | Application database |
| Redis | `Deployment` + `Service` | Cache and task broker |
| Secrets | `Secret` | Passwords, Django secret key, encryption key, licence |
| Application | `Application` (app.k8s.io/v1beta1) | Groups the components in the Google Cloud console |

The DQE-One `Service` is of type `ClusterIP`: the application is not exposed outside the cluster until you configure an Ingress (see [Expose DQE-One on your own domain](#expose-dqe-one-on-your-own-domain)).

# Installation

## Quick install with Google Cloud Marketplace

Install DQE-One on a Google Kubernetes Engine cluster from the [Google Cloud Marketplace](https://console.cloud.google.com/marketplace) by following the on-screen instructions. All secrets are generated automatically.

## Command-line instructions

### Prerequisites

#### Tools

- [gcloud](https://cloud.google.com/sdk/gcloud/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- `envsubst` (package `gettext`), `openssl`, `git`

You can use [Cloud Shell](https://cloud.google.com/shell), where all these tools are already installed.

#### Create a GKE cluster

Skip this step if you already have a cluster.

```shell
export CLUSTER=dqe-one-cluster
export ZONE=europe-west1-b

gcloud container clusters create "$CLUSTER" \
  --zone "$ZONE" \
  --num-nodes 1 \
  --machine-type e2-standard-4
```

#### Configure kubectl to connect to the cluster

```shell
gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
```

#### Clone this repository

```shell
git clone https://github.com/dqe-software/dqe-one-gcp-marketplace.git
cd dqe-one-gcp-marketplace
```

#### Install the Application resource definition

The `Application` custom resource groups all the components of DQE-One in the Google Cloud console. Install its definition once per cluster:

```shell
kubectl apply -f "https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml"
```

### Install the application

#### Configure the application instance

Choose an instance name and a namespace:

```shell
export name=dqe-one-1
export namespace=default
```

Set the application version and the container images. Use a version track (for example `1.0`) to receive security patches automatically when you redeploy, or a full version (for example `1.0.0`) to pin a release:

```shell
export appVersion=1.0.0
export TAG=1.0

export imageDqeOne="marketplace.gcr.io/dqe-one-public/dqe-one/standalone:${TAG}"
export imagePostgresql="marketplace.gcr.io/dqe-one-public/dqe-one/postgresql:${TAG}"
export imageRedis="marketplace.gcr.io/dqe-one-public/dqe-one/redis:${TAG}"
```

> For production, we recommend referencing the images by digest. Resolve the digest of each image with:
>
> ```shell
> docker pull "$imageDqeOne" && docker inspect --format='{{index .RepoDigests 0}}' "$imageDqeOne"
> ```

Set the application settings:

```shell
# Superuser created on first startup
export adminUser=admin

# Your DQE-One licence key (provided by DQE Software)
export dqeClientLicence="YOUR_LICENCE_KEY"

# Domain your users will use, for example https://dqe.my-company.com
# Leave empty if you only access DQE-One through kubectl port-forward
export appDomain=""

# Size of the PostgreSQL persistent disk
export postgresqlDiskSize=20Gi
```

#### Generate the secrets

Passwords and keys are generated locally and never leave your machine. The first three values are base64-encoded because they are stored in the `data` section of the Kubernetes `Secret`:

```shell
export adminPassword=$(openssl rand -hex 10 | tr -d '\n' | base64 | tr -d '\n')
export dbPassword=$(openssl rand -hex 16 | tr -d '\n' | base64 | tr -d '\n')
export djangoSecretKey=$(openssl rand -hex 25 | tr -d '\n' | base64 | tr -d '\n')

# Fernet encryption key: 32 random bytes, URL-safe base64 (44 characters)
export encryptionKey=$(openssl rand -base64 32 | tr '+/' '-_' | tr -d '\n')
```

> ⚠️ Keep `encryptionKey` safe: data encrypted by DQE-One cannot be read without it. If you reinstall the application on an existing database, reuse the same key.

#### Create the namespace

Skip this step if you use the `default` namespace:

```shell
kubectl create namespace "$namespace"
```

#### Expand the manifest template

Only the variables listed below are substituted:

```shell
VARS='${name} ${namespace} ${imageDqeOne} ${imagePostgresql} ${imageRedis} ${adminUser} ${adminPassword} ${dqeClientLicence} ${appDomain} ${postgresqlDiskSize} ${dbPassword} ${djangoSecretKey} ${encryptionKey}'

for f in manifest/*.yaml; do echo "---"; cat "$f"; done \
  | sed "s/__RELEASE_VERSION__/${appVersion}/g" \
  | envsubst "$VARS" \
  > "${name}_manifest.yaml"
```

> `${name}_manifest.yaml` contains your secrets in clear text. Store it securely or delete it after installation.

#### Apply the manifest to your cluster

```shell
kubectl apply -f "${name}_manifest.yaml" --namespace "$namespace"
```

#### Check the deployment

```shell
kubectl get pods --namespace "$namespace" --watch
```

After 2 to 3 minutes, all pods should be `Running` with `READY 1/1`:

```
dqe-one-1-dqeone-xxxxxxxxxx-xxxxx   1/1   Running
dqe-one-1-postgresql-0              1/1   Running
dqe-one-1-redis-xxxxxxxxxx-xxxxx    1/1   Running
```

On first startup, DQE-One runs its database migrations: the `dqeone` pod can take up to 2 minutes to become ready.

To view the application in the Google Cloud console, open **Kubernetes Engine > Applications**.

# Access DQE-One

### From your workstation

```shell
kubectl port-forward --namespace "$namespace" "svc/${name}-dqeone" 8001:8001
```

Then open [http://localhost:8001](http://localhost:8001).

### Administrator credentials

```shell
kubectl get secret "${name}-secrets" --namespace "$namespace" \
  -o jsonpath='{.data.admin-user}' | base64 -d; echo
kubectl get secret "${name}-secrets" --namespace "$namespace" \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

# Expose DQE-One on your own domain

DQE-One expects HTTPS to be terminated in front of the application (Ingress or Load Balancer), which must forward the `X-Forwarded-Proto` header.

1. Set `appDomain` to your public URL (for example `https://dqe.my-company.com`) and re-apply the manifest (see [Update](#update)). This allows the domain in Django (`ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS`).
2. Reserve a global static IP address and create a DNS `A` record for your domain pointing to it:

   ```shell
   gcloud compute addresses create "${name}-ip" --global
   gcloud compute addresses describe "${name}-ip" --global --format='value(address)'
   ```

3. Create a Google-managed certificate and an Ingress. Example for GKE:

   ```shell
   export DOMAIN=dqe.my-company.com

   cat <<EOF | kubectl apply --namespace "$namespace" -f -
   apiVersion: networking.gke.io/v1
   kind: ManagedCertificate
   metadata:
     name: ${name}-cert
   spec:
     domains:
       - ${DOMAIN}
   ---
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: ${name}-ingress
     annotations:
       kubernetes.io/ingress.global-static-ip-name: ${name}-ip
       networking.gke.io/managed-certificates: ${name}-cert
       kubernetes.io/ingress.class: gce
   spec:
     defaultBackend:
       service:
         name: ${name}-dqeone
         port:
           number: 8001
   EOF
   ```

   Provisioning the certificate can take up to 60 minutes after the DNS record is in place.

4. Check that the GKE load balancer health check uses the `/login` path on port 8001. If the backend stays unhealthy, create a `BackendConfig` with a health check on `/login` and attach it to the service with the `cloud.google.com/backend-config` annotation.

# Update

To update the configuration or upgrade to a new version, change the relevant variables (for example `appVersion` and `TAG`) and **reuse the same secrets**: export the current values from the cluster instead of generating new ones:

```shell
get_secret() { kubectl get secret "${name}-secrets" --namespace "$namespace" -o jsonpath="{.data.$1}"; }

export adminPassword=$(get_secret admin-password)
export dbPassword=$(get_secret db-password)
export djangoSecretKey=$(get_secret secret-key)
export encryptionKey=$(get_secret encryption-key | base64 -d)
export adminUser=$(get_secret admin-user | base64 -d)
export dqeClientLicence=$(get_secret licence | base64 -d)
```

Then [expand the manifest](#expand-the-manifest-template) again and apply it with `kubectl apply`.

# Uninstall

### Using the Google Cloud console

1. Open **Kubernetes Engine > Applications**.
2. Select your DQE-One instance and click **Delete**.

### Using the command line

```shell
kubectl delete -f "${name}_manifest.yaml" --namespace "$namespace"
```

If you no longer have the expanded manifest, delete the resources by label:

```shell
kubectl delete application,deployment,statefulset,service,secret \
  --namespace "$namespace" \
  --selector "app.kubernetes.io/name=${name}"
```

### Delete the persistent volumes

The PostgreSQL disk is **not** deleted with the application, to protect your data. To delete it permanently:

```shell
kubectl delete persistentvolumeclaims \
  --namespace "$namespace" \
  --selector "app.kubernetes.io/name=${name},app.kubernetes.io/component=postgresql"
```

### Delete the GKE cluster

If you created a cluster only for DQE-One:

```shell
gcloud container clusters delete "$CLUSTER" --zone "$ZONE"
```

# Support

For licensing and support, contact [DQE Software](https://www.dqe-software.com).
