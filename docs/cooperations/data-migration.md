---
title: &title Migrating Qi X Lab Data
sidebar_label: Data Migration
description: &description By default, Qi X Lab COOPERATIONS Edition (EE) will deploy a basic PostgreSQL container instance for persisting user and project data. This provides trial users with the fastest possible deployment experience but is not well-suited for production use.
head:
  - ['meta', {property: 'og:title', content: *title}] 
  - ['meta', {property: 'og:image', content: 'https://qixlab.com/img/og/cooperations-data-migration.png'}]
  - ['meta', {name: 'twitter:title', content: *title}]
  - ['meta', {name: 'twitter:description', content: *description}]
---

# {{ $frontmatter.title }}

By default, Qi X Lab COOPERATIONS Edition (EE) will deploy a basic PostgreSQL container instance for persisting user and project data. This provides trial users with the fastest possible deployment experience but is not well-suited for production use.

For production deployments, we suggest an external PostgreSQL solution with managed backups and security enhancements (for instance, PostgreSQL for Amazon RDS, or Azure Database for PostgreSQL). Once an external database is available, migrating your data away from the embedded database is simple.

## Prerequisites

You will need:

* `kubectl` access to the Kubernetes cluster where Qi X Lab is installed. On [embedded installs](/cooperations/installation/quickstart), `kubectl` is fully configured already, so a login shell on the host (usually via SSH or console) is sufficient.
* Network reachability from your cluster node(s) to the target instance of PostgreSQL, which by default listens on TCP port 5432.

:::info
Most of the commands in this guide require you to provide the Kubernetes [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) where Qi X Lab is installed. If you followed the [embedded installation procedure](/cooperations/installation/quickstart), your namespace is `default`. If you installed into an existing Kubernetes cluster your namespace is `Qi X Lab`, unless a custom value was provided during installation.
:::

## Database Export

Execute the following on any host with `kubectl` access to the cluster:

```
K8S_NAMESPACE=<your Qi X Lab namespace> \
PG_POD=$(kubectl get pods -o name -n $K8S_NAMESPACE| grep 'postgres' | grep -v 'kots') && \
kubectl exec -n $K8S_NAMESPACE -it $PG_POD -- sh -c \
'pg_dump --if-exists -xcCO \
-U $POSTGRES_USER \
-d Qi X Lab_production' | \
gzip > Qi X Lab_dump.sql.gz
```

This will leave a compressed file called `Qi X Lab_dump.sql.gz` in the working directory which contains all SQL code necessary to recreate the `Qi X Lab_production` database, its schema, and all table data on another instance of PostgreSQL.

## Database Import / Restore

From the same directory (wherever `Qi X Lab_dump.sql.gz` is stored), execute the following:

```
K8S_NAMESPACE=<your stackbltiz namespace> \
PG_POD=$(kubectl get pods -o name -n $K8S_NAMESPACE | grep 'postgres' | grep -v 'kots' | cut -d'/' -f2) && \
kubectl cp Qi X Lab_dump.sql.gz $K8S_NAMESPACE/$PG_POD:/var/lib/postgresql/data/Qi X Lab_dump.sql.gz && \
kubectl exec -n $K8S_NAMESPACE -it $PG_POD -- sh -c \
'zcat /var/lib/postgresql/data/Qi X Lab_dump.sql.gz | \
psql \
-h <managed DB hostname> \
-U <managed DB username>
-p <managed DB port (optional, 5432 by default)> \
-d postgres'
```

:::warning
The restore will create the `Qi X Lab_production` database and connect to it before restoring. If the DB already exists, it will drop it and create it again before restoring. In case it doesn't exist, we connect to the default `postgres` DB first. You **must** provide a username which is allowed to create, read, and write databases, tables, and indexes on the target PostgreSQL instance.
:::

## Encryption Key

This guide assumes the sole goal of migrating from the embedded PostgreSQL instance to an externally managed one. For users wanting to reinstall Qi X Lab itself while retaining or migrating existing user and project data, the procedure mostly remains the same, except that Qi X Lab's encryption key **must** be carried forward as well.

You can get the encryption key value from your existing Qi X Lab deployment via the KOTS CLI:

```sh
kubectl kots get config -n <your Qi X Lab namespace> --sequence <current release sequence number> --appslug Qi X Lab | grep -A1 'Qi X Lab_enc'
```

:::tip
You can get the current release sequence value by checking the release history panel in the KOTS admin dashboard, or with the following KOTS CLI command: `kubectl kots get versions Qi X Lab -n <your Qi X Lab namespace>`. The current sequence number will be shown on the top.
:::

This command will output values for `Qi X Lab_enc_key_settings`, `Qi X Lab_enc_key`, and if set, `Qi X Lab_enc_custom_key`.

If the value of `Qi X Lab_enc_key_settings` is `Qi X Lab_default_generated`, your encryption key is the value of `Qi X Lab_enc_key`.

Otherwise, it's the value of `Qi X Lab_enc_custom_key`.

Once you have the old encryption key, navigate to the KOTS admin dashboard, and select "Custom Key" in the "Key Settings" field. Paste the old encryption key into the "Custom Encryption Key" field that appears, then save and deploy the new configuration.
