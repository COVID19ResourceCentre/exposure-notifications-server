# Exposure Notification Reference Server

## Building and deploying services

This page explains how to build and deploy servers within the Exposure
Notification Reference implementation.

### Before you begin

To build and deploy a service, you need to install and configure the following:

1. Download and install the [Google Cloud SDK](https://cloud.google.com/sdk/install).

    For more information on installation and to set up, see the
    [Cloud SDK Quickstarts](https://cloud.google.com/sdk/docs/quickstarts).

1. Download and install [Go 1.14.0 or newer](https://golang.org/dl/).

    Make sure the `go/bin/` folder is set in your `PATH` environment variable.
    For more information on installing and configuring Go, see
    [Install the Go tools](https://golang.org/doc/install#install).

1. Enable Go modules and install the `ko` container builder and deployment tool:

    ```console
    GO111MODULE=on
    go get github.com/google/ko/cmd/ko
    ```

1. Configure the `ko` tool using the `setup_ko.sh` configuration file:

    ```console
    source setup_ko.sh
    ```

### Building and deploying

To build and deploy a service:

1. Generate a [Google Cloud Repository](https://cloud.google.com/container-registry)
   Docker configuration:

    ```console
    gcloud auth configure-docker
    ```

1. Build and deploy the container using the `ko publish` command from the repository's
   root directory.

    For example, to deploy the exposure key server:

    ```console
    ko publish ./cmd/exposure
    ```

You can find a list of services and their corresponding folders below.

### List of services

The Exposure Notification Reference implementation includes multiple services.
Each service's `main` package is located in the `cmd` directory.

| Service | Folder                | Description |
|---------|-----------------------|-------------|
| exposure key server  | cmd/export | Publishes exposure keys |
| federation | cmd/federation | gRPC federation requests listener |
| federation puller | cmd/federation-pull | Pulls federation results from federation partners |
| exposure server | cmd/exposure |  Stores infection keys |
| exposure cleanup | cmd/cleanup-exposure | Deletes old exposure keys |
| export cleanup | cmd/cleanup-export | Deletes old exported files published by the exposure key export service |

### Provisioning infrastructure with Terraform

You can use [Terraform](https://www.terraform.io) to provision the initial
infrastructure, database, service accounts, and first deployment of the services
on Cloud Run. **Terraform does not manage the Cloud Run services after their
initial creation!**

See [Deploying with Terraform](../terraform/README.md) for more information.

### Deploying services

While Terraform does an initial deployment of the services, it does not manage
the Cloud Run services beyond their initial creation. If you make changes to the
code, you will need to build, deploy, and promote new services. The general
order of operations is:

1.  **Build** - this is the phase where the code is bundled into a container
    image and pushed to a registry.

1.  **Deploy** - this is the phase where the container image is deployed onto
    Cloud Run, but is not receiving any traffic.

1.  **Promote** - this is the phase where a deployed container image begins
    receiving all or a percentage of traffic.

#### Building

Build new services by using the script at `./scripts/build`, specifying the
following values:

-   `PROJECT_ID` (required) - your Google Cloud project ID.

-   `SERVICES` (required) - comma-separated list of names of the services to
    build, or "all" to build all. See the list of services in the table above.

-   `TAG` (optional) - tag to use for the images. If not specified, it uses a
    datetime-based tag of the format YYYYMMDDhhmmss.

```text
PROJECT_ID="my-project" \
SERVICES="export" \
./scripts/build
```

Expect this process to take 3-5 minutes.

#### Deploying

Deploy already-built container using the script at `./scripts/deploy`,
specifying the following values:

-   `PROJECT_ID` (required) - your Google Cloud project ID.

-   `REGION` (required) - region in which to deploy the services.

-   `SERVICES` (required) - comma-separated list of names of the services to
    deploy, or "all" to deploy all. Note, if you specify multiple services, they
    must use the same tag.

-   `TAG` (required) - tag of the deployed image (e.g. YYYYMMDDhhmmss).

```text
PROJECT_ID="my-project" \
REGION="us-central1" \
SERVICES="export" \
TAG="20200521084829" \
./scripts/deploy
```

Expect this process to take 1-2 minutes.

#### Promoting

Promote an already-deployed service to begin receiving production traffic using
the script at `./scripts/promote`, specifying the following values:

-   `PROJECT_ID` (required) - your Google Cloud project ID.

-   `REGION` (required) - region in which to promote the services.

-   `SERVICES` (required) - comma-separated list of names of the services to
    promote, or "all" to deploy all. Note, if you specify multiple services,
    then the revision must be "LATEST".

-   `REVISION` (optional) - revision of the service to promote, usually the
    output of a deployment step. Defaults to "LATEST".

-   `PERCENTAGE` (optional) - percent of traffic to shift to the new revision.
    Defaults to "100".

```text
PROJECT_ID="my-project" \
REGION="us-central1" \
SERVICES="export" \
./scripts/promote
```

Expect this process to take 1-2 minutes.

### Running migrations

#### On Google Cloud

To migrate the production database, use the script in `./scripts/migrate`. This
script triggers a Cloud Build invocation which uses the Cloud SQL Proxy to run
the database migrations and uses the following environment variables:

-   `PROJECT_ID` (required) - your Google Cloud project ID.

-   `DB_CONN` (required) - your Cloud SQL connection name.

-   `DB_PASS_SECRET` (required) - the **reference** to the secret where the
    database password is stored in Secret Manager.

-   `DB_NAME` (default: "main") - the name of the database against which to run
    migrations.

-   `DB_USER` (default: "notification") - the username with which to
    authenticate.

-   `COMMAND` (default: "up") - the migration command to run.

If you created the infrastructure using Terraform, you can get these values by
running `terraform output` from inside the `terraform/` directory:

```text
PROJECT_ID=$(terraform output project)
DB_CONN=$(terraform output db_conn)
DB_PASS_SECRET=$(terraform output db_pass_secret)
```

#### On a custom setup

If you did not use the Terraform configurations to provision your server, or if you are running your own Postgres server,

1.  Download and install the
    [`migrate`](https://github.com/golang-migrate/migrate) tool.

1.  Construct the [database URL](https://github.com/golang-migrate/migrate/tree/master/database/postgres) for your database. This is usually of the format:

    ```text
    postgres://DB_USER:DB_PASSWORD@DB_HOST:DB_PORT/DB_NAME?sslmode=require
    ```

1.  Run the migrate command with this database URL:

    ```text
    migrate \
      -database "YOUR_DB_URL" \
      -path ./migrations \
      up
    ```
