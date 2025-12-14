---
title: 'Deploy Phoenix to Google Cloud Run and Cloud SQL'
description: 'Learn how to set up automated deployments and migrations for a Phoenix project onto Google Cloud Run and Cloud SQL.'
published: 2024-01-26
draft: false
tags: ['gcp', 'elixir']
toc: true
---

This post explains how to set up **automated deployments and migrations** for a Phoenix project on Google Cloud's managed services using the [Google Cloud CLI](https://cloud.google.com/sdk/gcloud) (mostly). The Phoenix app will be hosted on Google Cloud Run and the PostgreSQL database will be hosted on Cloud SQL. Deployments will be automatically triggered when changes are pushed to the `main` branch of your git repository (GitHub specifically in this post).

At a high level we will:

1. Prepare your application
2. Create a GCP project
3. Enable the services we need
4. Create an Artifact Registry repository to store our compiled app
5. Update Service Account permissions
6. Create a Cloud SQL database instance
7. Create environment variables in Secrets Manager
8. Connect a GitHub repository to Cloud Build
9. Create a Cloud Build trigger
10. Create a build configuration file
11. Trigger a deploy to Cloud Run
12. (OPTIONAL) psql into Cloud SQL

## Prerequisites

- A Google account
- The [Google Cloud CLI](https://cloud.google.com/sdk/gcloud) installed and logged in
- A billing account set up on your Google Cloud organisation
- A Phoenix app

:::tip
Follow along with a generated Phoenix app
:::

```sh
mix phx.new insight
cd insight
# create something for us to test DB interaction with e.g.,
mix phx.gen.live Products Product products name brand
# remember to update lib/insight_web/router.ex
```

## 1. Prepare your application

Generate a Dockerfile and other useful release helpers.

```sh
mix phx.gen.release --docker
```

Delete the database environment variables in the runtime config.

```diff lang="elixir" title="config/runtime.exs"
if config_env() == :prod do
-  database_url =
-    System.get_env("DATABASE_URL") ||
-      raise """
-      environment variable DATABASE_URL is missing.
-      For example: ecto://USER:PASS@HOST/DATABASE
-      """
-
  maybe_ipv6 = if System.get_env("ECTO_IPV6") in ~w(true 1), do: [:inet6], else: []

  config :insight, Insight.Repo,
    # ssl: true,
-    url: database_url,
    pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
    socket_options: maybe_ipv6
```

:::note
`Postgrex.start_link/1` [(docs)](https://hexdocs.pm/postgrex/Postgrex.html#start_link/1) defaults to [PostgreSQL environment variables](https://www.postgresql.org/docs/current/libpq-envars.htm) (e.g., `PGHOST`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`) when no database connection details are specified.
:::

Cloud Run automatically generates a semi-randomised URL for your app (E.g., `https://[SERVICE NAME]-[RANDOM NUMBERS].a.run.app`). To prevent infinite reloading behaviour in LiveView we need to allow-list the Cloud Run origin.

```diff lang="elixir" title="config/prod.exs"
config :insight, Insight.Endpoint,
  cache_static_manifest: "priv/static/cache_manifest.json",
+  check_origin: ["https://*.run.app"]
```

## 2. Create a GCP project

:::tip
Use environment variables so that most commands can be copy-pasted. 
:::

Create environment variables for your application name and preferred [GCP region](https://cloud.google.com/compute/docs/regions-zones#available).

```sh
APP_NAME=insight
SERVICE_NAME=insight-dev
REGION=australia-southeast1

echo $APP_NAME $SERVICE_NAME $REGION
```

Create a project. Accept the auto-generated project ID at the prompt.

```sh
gcloud projects create --set-as-default --name="$SERVICE_NAME"
```

Create environment variables for your project ID and project number. It may take ~30 seconds before the project successfully returns with the `gcloud projects list` command.

```sh
PROJECT_ID=$(gcloud projects list \
    --format='value(projectId)' \
    --filter="name='$SERVICE_NAME'")

PROJECT_NUMBER=$(gcloud projects list \
    --format='value(projectNumber)' \
    --filter="name='$SERVICE_NAME'")

echo $PROJECT_ID $PROJECT_NUMBER
```

Find the billing account you set up (refer to prerequisites).

```sh
gcloud billing accounts list
```

Add your billing account from the prior command to the project.

```sh
gcloud billing projects link $PROJECT_ID \
    --billing-account=[billing_account_id]
```

## 3. Enable the services we need

Google Cloud disables all cloud products/services on a new project by default so we will need to enable all the services we will use for this deployment: Artifact Registry, Cloud Build, Cloud SQL, Secret Manager, Cloud Run, and the IAM API.

The following command will enable all the services we need.

```sh
gcloud services enable \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  compute.googleapis.com \
  sqladmin.googleapis.com \
  secretmanager.googleapis.com \
  run.googleapis.com \
  iam.googleapis.com
```

## 4. Create an Artifact Registry repository to store our compiled app

Create a new repository with an identifier (I generally align this with my elixir app name) and specifying the format and region.

```sh
gcloud artifacts repositories create $APP_NAME \
  --repository-format=docker \
  --location=$REGION \
  --description="$APP_NAME application"
```

Create an environment variable to capture the repository's `Registry URL` with the following command:

```sh
REGISTRY_URL=$(gcloud artifacts repositories describe $APP_NAME \
  --location $REGION)

echo $REGISTRY_URL
```

## 5. Update service account permissions

Update the service account to have all necessary permissions.

- `roles/logging.logWriter` permissions are required by Cloud Build
- `roles/cloudsql.client` permissions are required to interact with Cloud SQL
- `roles/artifactregistry.writer` permissions are required to read/write to Artifact Registry
- `roles/run.developer` permissions are required to deploy on Cloud Run
- `roles/iam.serviceAccountUser` permissions are required to allow the Service Account to "act as" another service account and assign ownership of services (such as Cloud Run). In this case the account is acting as itself, but it is still required despite being self-referential

Above can be added with the following commands:

```sh
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role="roles/logging.logWriter" \
  --condition None

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role="roles/cloudsql.client" \
  --condition None

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role="roles/artifactregistry.writer" \
  --condition None
```

## 6. Create a Cloud SQL database instance

Set up the environment variables for the next handful of steps.

```sh
INSTANCE=${APP_NAME}-sql
DB_NAME=${APP_NAME}_dev
DB_USER=${APP_NAME}_admin
DB_PASS=pa55w0rd

echo $INSTANCE $DB_NAME $DB_USER $DB_PASS
```

:::important
The instance name must be composed of lowercase letters, numbers, and hyphens; must start with a letter.
:::

Create a new PostgreSQL instance specifying your desired region, type of DB, and compute tier. This can take a few minutes to provision.

```sh
gcloud sql instances create $INSTANCE \
  --region=$REGION \
  --database-version=POSTGRES_17 \
  --tier=db-perf-optimized-N-2
```

:::note
We've used the lowest tier for this example, [refer to the pricing](https://cloud.google.com/sql/pricing#2nd-gen-instance-pricing).
:::

Create a user for the application to use when interacting with the database.

```sh
gcloud sql users create $DB_USER \
  --instance=$INSTANCE \
  --password=$DB_PASS
```

Create the database.

```sh
gcloud sql databases create $DB_NAME \
  --instance $INSTANCE
```

Retrieve the instance `connectionName` for later:

```sh
CONN_NAME=$(gcloud sql instances describe $INSTANCE \
  --format='value(connectionName)')

echo $CONN_NAME
```

## 7. Create environment variables in Secrets Manager

Create the following secrets on GCP that our Phoenix app will use:

- `DEV_SECRET_KEY_BASE` (mapped to SECRET_KEY_BASE in deploy step)
- `DB_USER` (mapped to PGUSER in deploy step)
- `DB_PASS` (mapped to PGPASSWORD in deploy step)
- `DB_HOST` (mapped to PGHOST in deploy step)

```sh
mix phx.gen.secret | gcloud secrets create DEV_SECRET_KEY_BASE --data-file=-
echo /cloudsql/$CONN_NAME | gcloud secrets create DB_HOST --data-file=-
echo $DB_USER | gcloud secrets create DB_USER --data-file=-
echo $DB_PASS | gcloud secrets create DB_PASS --data-file=-
```

Add permissions to the Service Account so it can access the secrets.

```sh
gcloud secrets add-iam-policy-binding DEV_SECRET_KEY_BASE \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding DB_USER \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding DB_PASS \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding DB_HOST \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role="roles/secretmanager.secretAccessor"
```

Retrieve the paths for the `DB_USER` and `DB_PASS` for use later:

```sh
DB_USER_PATH=$(gcloud secrets describe DB_USER --format='value(name)')
DB_PASS_PATH=$(gcloud secrets describe DB_PASS --format='value(name)')

echo $DB_USER_PATH $DB_PASS_PATH
```

## 8. Connect a GitHub repository to Cloud Build

This step and the next step are easier via [Google Cloud Console > Cloud Build > Repositories](https://console.cloud.google.com/cloud-build/repositories).

Click "CREATE HOST CONNECTION" and populate the fields.

It will then take you through authentication with GitHub. You will have an option to provide access to all of your GitHub repositories or just a selection. Pick whatever makes sense for your needs.

After you have successfully created a connection, click "LINK A REPOSITORY". Select the connection you just created and your Phoenix app repository. Choose generated repository names.

## 9. Create a Cloud Build trigger

Create a trigger via [Google Cloud Console > Cloud Build > Triggers](https://console.cloud.google.com/cloud-build/triggers).

Click "CREATE TRIGGER" and populate with your desired details:

- Event: Push to a branch
- Source: 2nd gen
- Repository: Select the one you linked in prior step
- Branch: Will auto populate with a regular expression to match the main branch `^main$`
- Type: Cloud Build configuration file
- Location: Repository
- Cloud Build configuration file location: `/cloudbuild.yaml`

## 10. Create a build configuration file

In your Phoenix project's root directory run the following command to create a `cloudbuild.yaml` file.

```sh
cat << EOF > cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  id: Build and Push Docker Image
  script: |
    docker build -t \${_IMAGE_NAME}:latest .
    docker push \${_IMAGE_NAME}:latest

- name: 'gcr.io/cloud-builders/docker'
  id: Start Cloud SQL Proxy to Postgres
  args: [
      'run',
      '-d',
      '--name',
      'cloudsql',
      '-p',
      '5432:5432',
      '--network',
      'cloudbuild',
      'gcr.io/cloud-sql-connectors/cloud-sql-proxy',
      '--address',
      '0.0.0.0',
      '\${_INSTANCE_CONNECTION_NAME}'
    ]

- name: 'postgres'
  id: Wait for Cloud SQL Proxy to be available
  script: |
    until pg_isready -h cloudsql ; do sleep 1; done

- name: \${_IMAGE_NAME}:latest
  id: Run migrations
  env:
  - MIX_ENV=prod
  - SECRET_KEY_BASE=fake-key
  - PGHOST=cloudsql
  - PGDATABASE=\${_DATABASE_NAME}
  secretEnv:
  - PGUSER
  - PGPASSWORD
  script: |
    /app/bin/insight eval "Insight.Release.migrate"

- name: 'gcr.io/cloud-builders/gcloud'
  id: Deploy to Cloud Run
  args: [
    'run',
    'deploy',
    '\${_SERVICE_NAME}',
    '--image',
    '\${_IMAGE_NAME}:latest',
    '--region',
    '\${_REGION}',
    '--allow-unauthenticated',
    '--set-secrets=SECRET_KEY_BASE=DEV_SECRET_KEY_BASE:latest',
    '--set-secrets=PGHOST=DB_HOST:latest',
    '--set-secrets=PGUSER=DB_USER:latest',
    '--set-secrets=PGPASSWORD=DB_PASS:latest',
    '--set-env-vars=PGDATABASE=\${_DATABASE_NAME}',
    '--add-cloudsql-instances=\${_INSTANCE_CONNECTION_NAME}'
  ]

availableSecrets:
  secretManager:
  - versionName: $DB_USER_PATH/versions/latest
    env: 'PGUSER'
  - versionName: $DB_PASS_PATH/versions/latest
    env: 'PGPASSWORD'

images:
  - \${_IMAGE_NAME}:latest

options:
  automapSubstitutions: true
  logging: CLOUD_LOGGING_ONLY

substitutions:
  _DATABASE_NAME: $DB_NAME
  _IMAGE_NAME: $REGISTRY_URL/app
  _INSTANCE_CONNECTION_NAME: $CONN_NAME
  _REGION: $REGION
  _SERVICE_NAME: insight-dev
EOF
```

This script creates a `cloudbuild.yaml` [(docs)](https://cloud.google.com/build/docs/build-config-file-schema) that:

  - Builds our application image and pushes it to Artifact Repository
  - Starts a Cloud SQL Proxy within the Cloud Build environment
  - Waits to ensure the proxy is functional
  - Executes up migrations against the database
    - Despite using `MIX_ENV=prod` we are still interacting with the database we created previously (`insight_dev`) via the `PGDATABASE` environment variable
    - The migrations are run using the scripts generated by `mix phx.gen.release --docker`
    - Uses our image and the PostgreSQL environment variables (`PGHOST`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`)
  - Deploys our Cloud Run service
    - Uses our image
    - Maps the secrets and environment variables
    - Assigns our service account as the owner
    - Links our Cloud SQL instance
- Makes use of [substitute variables](https://cloud.google.com/build/docs/configuring-builds/substitute-variable-values) to make it easier to work with. Because we are using a mix of `script:` and `arg:` approaches we need to set the `automapSubstitutions: true` option otherwise our builds will fail

## 11. Trigger a deploy to Cloud Run

Commit the `cloudbuild.yaml` file (or any other change) and push it to your GitHub repository and watch it build. You can manually trigger builds via [Google Cloud Console > Cloud Build > Triggers](https://console.cloud.google.com/cloud-build/triggers).

You can view previous builds and stream in-progress builds on the [Cloud Build History tab](https://console.cloud.google.com/cloud-build/builds).

You should now have a fully deployed application on GCP!

If at any time you need to retrieve details of this service you can do so with the following command

```sh
gcloud run services list
```

## 12. (OPTIONAL) psql into Cloud SQL

To remotely connect to the Cloud SQL database you can use Cloud SQL Proxy. This securely connects via API to the database using your Google Cloud CLI credentials.

Download and install the [Cloud SQL Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy#install). Follow the instructions at the link.

Cloud SQL Proxy uses Google Cloud CLI credentials for auth, set them with:

```sh
gcloud auth application-default login
```

Start the proxy using `connectionName`. The port must not already be in use.

```sh
./cloud-sql-proxy --port 54321 $CONN_NAME
```

If successful you will see see output similar to:

```sh
Authorizing with Application Default Credentials
Listening on 127.0.0.1:54321
```

Now you can psql in!

```sh
psql host="127.0.0.1 port=54321 sslmode=disable user=$DB_USER dbname=$DB_NAME"
```

