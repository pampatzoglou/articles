### **Into (WIP)**

A journey into discovering how to handle state through databases in a production environment with examples that I found useful. We will consider several aspects:

# Table of Contents

- [Regarding Isolation](#regarding-isolation)
- [Regarding lifecycle management](#regarding-lifecycle-management)
- [Regarding security](#regarding-security)
- [Regarding Disaster recovery](#regarding-disaster-recovery)
- [Regarding scalability](#regarding-scalability)
- [Regarding observability](#regarding-observability)
- [Regarding developers](#regarding-developers)

# Regarding Isolation

Let's primerelaly focus on the isolation options for a database.

### **1. Shared Database, Shared Schema**

* All tenants use the same database and the same set of tables.
* Each entry includes a ``tenant_id`` column to segregate data.
* Efficient use of resources is debatable because of the `WHERE tenant_id` will add overhead to each query.
* **Pros:**

  * Simplifies deployment and maintenance.
* **Cons:**

  * Requires strict tenant-aware data access controls. If you implement a ROW-level ACL (implementation follows).
  * Certain performance bottlenecks as the tenant count grows.
  * Extremely bad If there is a DR scenario:
    * All users will be affected as you will need to rollback ALL the database to the last known good state.
    * botched operations will be global.
    * global recovery increases MTTR.
* **Notes:**
  One can use Row-Level Security (RLS) in Postgres to have some weak aggregated form of isolation, but its not isolation. We will dive into this subject as part of security.

> A bad idea here to shards tenants would be to use table prefixes. **DONT**.

### 2. **Shared Database, Separate Schema**

* Each tenant has its own schema within a shared database. Table names etc within the schema should be consistent to simplify development.
* Schema-based isolation improves security and performance. From a security perspective, this isolation is considered weak because you need to protect from:
  * Schema Privilege Escalation
  * Cross-Schema Injection Risks
* **Pros:**
  * Better sharding of data while sharing infrastructure.
  * Queries no longer include the `WHERE tenant_id` portion which adds overhead to queries.
  * Allows per-tenant schema customizations. Avoid if possible.
  * Future proof to move to more isolated setups.
* **Cons:**
  * More complex migrations and schema updates.
  * If not properly managed, restoring one schema could expose data from others due to shared connections.
  * Increased administrative overhead, but solvable.
  * Still need to have strict database logs and alerts to detect unauthorized access because all users will be working on the same process.
  * Complicated backups because each schema should have a dedicated backup process.
  * Can't use volume snapshots for backups. They will rever the global state meaning you affect all tenants.

### 3. **Separate Databases per Tenant, Shared database instance**

* Each tenant has its own dedicated database but all tenants use the same CPUs.
* Ensures complete data isolation.
* **Pros:**
  * All benefits for Shared Database, Separate Schema setup plus:
  * Strong ACL offers improved security.
  * Eases tenant-specific backups and compliance handling.
  * Simplest and fastest DR.
  * DR is not global.
* **Cons:**
  * Migration and schema updates are still complex.
  * Higher infrastructure and management overhead. Will discuss the solution later in this doc.
  * Efficient scaling needs more thought.

### 4. Separate databases instance per tenant

* Each tenant is running on their own database, probably in their dedicated namespace. Or perhaps on a different region etc. At this level of isolation, it doesn't matter.
* Even stronger isolation because you don't rely only on db-level ACL but also network policies/firewall rules.
* In essence it's a special case of the 3rd option with:
* **Pros:**

  * Even stronger isolation
* **Cons:**

  * Capacity planning and optimization overhead will be significant.
  * To benefit from this level you need to ensure that the undelaying infrastructure setup is extremely secure so your processes are not exposed to other people running on the same cloud infra eg
    ```yaml
      NodeLaunchTemplate:
        Type: AWS::EC2::LaunchTemplate
        Properties:
          LaunchTemplateData:
             MetadataOptions:
              HttpPutResponseHopLimit: 1
              HttpTokens: required

      EksCluster:
        Type: AWS::EKS::Cluster
        Properties:
          Name: !Sub "${AWS::StackName}"
          RoleArn: !GetAtt ClusterIamRole.Arn
          EncryptionConfig:
            - Provider: 
                KeyArn: !GetAtt ClusterSecretsKMSKey.Arn
              Resources:
              - secrets
          Logging:
            ClusterLogging:
              EnabledTypes:
                - Type: api
                - Type: audit
                - Type: authenticator
                - Type: controllerManager
                - Type: scheduler
          ResourcesVpcConfig:
            EndpointPublicAccess: false
            EndpointPrivateAccess: true
          KubeAPIServer:
            HTTPTokens: required
            HTTPPutResponseHopLimit: 1
    ```

In a kubernetes context you should be using different and unique security contexts per instance, dropped capabilities and with AppArmor or SeLinux, eg:

```yaml
---
  securityContext:
    runAsUser:1000
    runAsGroup:3000
    fsGroup:2000
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false

---
  securityContext:
    runAsUser:1001
    runAsGroup:3001
    fsGroup:2001
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false


```

# Regarding lifecycle management

Use two different jobs to handle the distinct aspects of the application lifecycle. The first Job runs first only on installation and its single responsibility is to create the database with the required parameters. In fact, it will run even before installation as we use the `pre-install` hook, which means before any application-specific resources are created.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "chart.fullname" . }}-db-create-{{ randAlphaNum 5 | lower }}
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-20"
    "helm.sh/hook-delete-policy": before-hook-creation
    "argocd.argoproj.io/hook": "PreSync"
    "argocd.argoproj.io/hook-delete-policy": "BeforeHookCreation"
    "argocd.argoproj.io/job-cleanup": "keep"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-create

```

and also define the migrate job that will run after the `db-create` and will setup the required schema for the application to function.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "chart.fullname" . }}-db-migrate-{{ randAlphaNum 5 | lower }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-10"
    "helm.sh/hook-delete-policy": before-hook-creation
    "argocd.argoproj.io/hook": "Sync"
    "argocd.argoproj.io/hook-delete-policy": "BeforeHookCreation"
    "argocd.argoproj.io/job-cleanup": "keep"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-migrate
```

the sequence of event should look something like this:

```mermaid
sequenceDiagram
    autonumber
    participant Helm as Helm Chart Installation
    participant ArgoCD as ArgoCD Controller
    participant K8s as Kubernetes
    participant DB as Database
    participant App as Application

    Helm->>ArgoCD: Apply Helm Chart (PreSync Hooks)
  
    %% Pre-Install Job: Database Creation %%
    ArgoCD->>K8s: Create Pre-Install Job (db-create)
    K8s->>DB: Initialize Database (Admin Credentials)
    DB-->>K8s: Database Created
    K8s-->>ArgoCD: Job db-create Completed

    %% Pre-Install & Pre-Upgrade Job: Schema Migration %%
    ArgoCD->>K8s: Create Pre-Install,Pre-Upgrade Job (db-migrate)
    K8s->>DB: Apply Schema Migrations (Admin Credentials)
    DB-->>K8s: Migrations Applied
    K8s-->>ArgoCD: Job db-migrate Completed

    %% Proceed with Application Deployment %%
    ArgoCD->>Helm: Proceed with Deployment
    Helm->>K8s: Deploy Application

    %% Application Runtime Using Limited Credentials %%
    K8s->>App: Deploy with Runtime Credentials
    App->>DB: Connect to Database (Runtime Credentials)
    DB-->>App: Access Granted

    App->>DB: Perform Read/Write Operations
    DB-->>App: Data Retrieved/Updated
```

# Regarding security

Security includes multiple aspects that we will approach.

## Access

In a nutshell access is who can knock on your door, meaning who can reach the port the database listens to. This might be enforced by security groups or network policies etc but in essence we are talking firewalls here.Following is an example of a NetworkPolicy (kubernetes firewall) that allows an app, the vault and prometheus to access it. Here we allow Prometheus running on monitoring namespace, the replicas from the same namespace, and vault from the vault namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-db-postgresql-primary-ingress
  labels:
    app.kubernetes.io/instance: app-db
    app.kubernetes.io/component: primary
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/instance: app-db
      app.kubernetes.io/name: postgresql
      app.kubernetes.io/component: primary
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/instance: app-db
              app.kubernetes.io/name: postgresql
              app.kubernetes.io/component: read
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: vault
          podSelector:
            matchLabels:
              app.kubernetes.io/instance: vault
              app.kubernetes.io/name: vault
      ports:
        - port: 5432
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - port: 9187

```

## Authentication

Authentication is what happens after someone knocks on your door (port). What is required is for them to proove that they are who they claim to be in an acceptable format. There are two main types of methods to do this but always remember that no matter which approach you select what you want is to have is a safe central place where these are controlled. If you choose to do a bit of mix a match, make sure that you have clear separation of method with a clear understanding of why. The worse thing you can do here is to leverage multiple ways to authenticate on the same resources because you will lose track. Also to avoid suprises make sure that your authentication methods will work for production requirements and developers. Personally I recommend using service authentication between "stuff" that are offered by your cloud provider and your cluster's workloads and credentials between your engineering teams, your applications and the databases.

### authentication through roles

Authentication through roles means that you either trust some services metadata or their serviceAccount to allow them to do a particular action.

### authentication through credential

Authentication though credentials means that the service or user can actually produce a username/password combination that allows you to connect. A bit of a heads up here is that its safer to use files as source of credentials for any application instead of using environmental variables for the simple reason that if someone runs `ps env` they will read the entire environment of the container and thus the credentials used. Also consider that dynamic credentials are far better than static ones. Using dynamic credentials means that there are no credentials to be saved in the code or reused though .env that might be shared between team members.

> Do not try and bend the spoon, that's impossible. Instead try to realise the truth, there is no spoon.

Now let's get dirty, in reality your application should have two procceses, one that handles the schema through migrations and a second for normal operation. This means that you will have some secrets in the cluster that will look like this:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "chart.fullname" . }}-db-migrate-credentials
type: Opaque
stringData:
  DB_USER: 
  DB_PASSWORD: 
  DB_NAME: 
---
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "chart.fullname" . }}-db-runtime-credentials
type: Opaque
stringData:
  DB_USER: 
  DB_PASSWORD: 
  DB_NAME: 

```

There are several approaches to facilitating this but here I will dive into my favourite.

#### Hashicorp vault

I consider Vault the root user of all databases. It gives access to all of them and with secret engines, eg [postgres](https://developer.hashicorp.com/vault/docs/secrets/databases/postgresql) it can generate [ephimeral credentials](https://developer.hashicorp.com/vault/docs/secrets/databases/postgresql).

These credentials can be mapped to IAM roles, or use [external secrets operator](https://external-secrets.io/latest/introduction/overview/) or vault secrets operator. Whichever approach in the end, a short-lived username/password is generated that accesses the single database or schema with the required permissions. Keep in mind that these should be different for the process that handles schema (which needs to have elevated access eg to create a table) and for the normal app/role that will be allowed to only read/write/update but never delete or drop. The beauty of this is that from the developer's perspective it's the same secret (though different values) that the application uses. So they will not need to bother with the overhead of this logic in the application code. The downside of this approach is that when values change, it will require that the pod gets restated which may cascade and have issues with PDB.

An example generate the aforementioned secrets using the vault and external secrets operator is:

```bash
vault write database/config/my-postgresql-database \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@your-db-host:5432/postgres?sslmode=disable" \
    allowed_roles="db-admin, runtime-user" \
    username="vaultuser" \
    password="vaultpassword"

vault write database/roles/db-admin \
    db_name=my-postgresql-database \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}'; GRANT ALL PRIVILEGES ON DATABASE mydb TO \"{{name}}\";" \
    revocation_statements="REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\"; DROP ROLE \"{{name}}\";" \
    default_ttl="5m" \
    max_ttl="1h"

vault write database/roles/runtime-user \
    db_name=my-postgresql-database \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}'; GRANT CONNECT ON DATABASE mydb TO \"{{name}}\"; GRANT USAGE ON SCHEMA public TO \"{{name}}\"; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    revocation_statements="REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\"; DROP OWNED BY \"{{name}}\"; DROP ROLE \"{{name}}\";"
    default_ttl="30m" \
    max_ttl="1h"
```

Then by using some facilitator service you can have dynamic secrets where they need to be. The ones most offtenly used are either vault agent that will use some annotations to setup the credentials for the pod or external secrets operator which will interact with the vault and generate the kubernetes sercret that will be used. Some more [examples](https://dev.to/breda/dynamic-postgresql-credentials-using-hashicorp-vault-with-php-symfony-go-examples-4imj)

```yaml
---
apiVersion: external-secrets.io/v1alpha1
kind: VaultProvider
metadata:
  name: vault-provider
spec:
  vault:
    address: "https://your-vault-address:8200"
    auth:
      tokenSecretRef:
        name: vault-token
        key: token
---
apiVersion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
  name: db-admin-credentials
spec:
  provider:
    vault:
      path: "database/creds/db-admin"  # Use the appropriate role for DB Admin or Runtime User
      server: "https://your-vault-address:8200"
      auth:
        tokenSecretRef:
          name: vault-token
          key: token
  secretStoreRef:
    name: vault-provider
  target:
    name: db-credentials
    creationPolicy: Owner
    data:
      - secretKey: DB_USER
        remoteRef:
          key: database/creds/db-admin
          property: data.username
      - secretKey: DB_PASSWORD
        remoteRef:
          key: database/creds/db-admin
          property: data.password
      - secretKey: DB_NAME
        remoteRef:
          key: database/creds/db-admin
          property: data.dbname
---
apiVersion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
  name: db-runtime-credentials
spec:
  provider:
    vault:
      path: "database/creds/db-runtime"  # Use the appropriate role for DB Admin or Runtime User
      server: "https://your-vault-address:8200"
      auth:
        tokenSecretRef:
          name: vault-token
          key: token
  secretStoreRef:
    name: vault-provider
  target:
    name: db-credentials
    creationPolicy: Owner
    data:
      - secretKey: DB_USER
        remoteRef:
          key: database/creds/runtime-user
          property: data.username
      - secretKey: DB_PASSWORD
        remoteRef:
          key: database/creds/runtime-user
          property: data.password
      - secretKey: DB_NAME
        remoteRef:
          key: database/creds/runtime-user
          property: data.dbname

```

The main problem is that by nature of vault etc, not all of these can be automated. It can be scripted but a person who actually has access to the vault will need to be involved to run the required configurations. Unless you create a custom operator that has admin vault access and performs the required actions to setup the particular engine and paths etc. A better approach is to use the vault agent

```yaml
...
  template:
    metadata:
      labels:
        app: sb-k8s-template
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-k8s-role"
        vault.hashicorp.com/agent-inject-secret-myapp-db: "myapp-db/creds/myapp-db-role"
        vault.hashicorp.com/agent-inject-file-secret-myapp-db: "myapp-db.creds"
        vault.hashicorp.com/auth-path: "auth/kubernetes"
        vault.hashicorp.com/agent-run-as-user: "1881"
        vault.hashicorp.com/agent-pre-populate: "true"
        vault.hashicorp.com/agent-pre-populate-only: "false"...
```

Which ever method you select, keep in mind that at the end it will mean that the client gets a token/key to use. Basically a passport that they will then use for authorization.

## Authorization

Authorization is what happens after a user or a proccess used their credentials or token and has some form of access. So after a connection has been established and authentication is complete, authorization answers the question: what can I do?

### Shared Database, Shared Schema

Previously we have mentioned Row-Level Security (RLS). RLS is a method to define in a shared database and shared tables, who can do what. In essence what you want is to try and block tenant1 from reading or writing entries that belong to tenant2. RLS is a way to atchive this result but at a computational cost for the database. For **each query** the database will need to check if the cursor has the required access to the particular row and then perform the actual query with the mentioned `WHERE tenant_id`. All this wll add delay.

```sql
--- Enable Row-Level Security (RLS) on the target table.
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

--- Create a Security Policy for Tenants (restrict access to own rows).
CREATE POLICY tenant_row_access
ON orders
FOR ALL
USING (tenant_id = current_setting('app.current_tenant')::UUID);

--- Ensure INSERT operations only allow the correct tenant_id.
CREATE POLICY tenant_insert_policy
ON orders
FOR INSERT
WITH CHECK (tenant_id = current_setting('app.current_tenant')::UUID);

--- Grant basic permissions on the orders table to tenant-specific roles.
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO tenant_role;

--- Set the tenant ID in the session (should be done by the application per request).
SELECT set_config('app.current_tenant', 'tenant-123', false);

--- Example: Tenant 123 querying the orders table (will only return their own rows).
SELECT * FROM orders;

--- Example: Trying to insert an order for another tenant (should fail).
INSERT INTO orders (id, tenant_id, order_name) VALUES (1, 'tenant-999', 'Test Order');
```

Nevertheless, even with RLS the two issues remain:

1. It's considered easy to jailbreak.
2. You will still need to create a migrations-like process that will generate and update the RLS as you add new tenants.

   I would suggest experimenting with using `EXPLAIN` and `EXPLAIN ANALYZE` on your data to see how it works on your system. Depending on scale, some organizations will be happy to accept the RLS overhead, and for some it will be a significant price that will drive them to adopt a different approach. From expirience what tends to happen is that organization start by using RLS and at some point start to split their clients into higher tiers where multitennancy is offered.

### Shared Database, Separate Schema or better

Once you have stopped using Shared Database with Shared Schema the authorization portion becomes rather simple and fast. This is because you how have access rules that are applied once when you start the cursor to the database and it will no longer need to be re calculated for each query the cursor makes.

# Regarding Disaster Recovery

**Disaster Recovery (DR)** is a set of policies, tools, and procedures designed to restore IT infrastructure and operations after a disaster (e.g., cyberattacks, hardware failures, natural disasters, or human errors). It ensures business continuity by minimizing downtime and data loss.

**Key Components of Disaster Recovery**

1. **Backup & Restore** – Regular data backups to on-site, off-site, or cloud storage.
2. **Disaster Recovery Plan (DRP)** – A documented strategy outlining recovery steps, responsibilities, and timelines.
3. **Recovery Time Objective (RTO)** – Maximum acceptable downtime before services must be restored.
4. **Recovery Point Objective (RPO)** – Maximum acceptable data loss measured in time (e.g., last backup timestamp).

In my opinion a common misconception is that HA setups offer DR. While they can be handy in some cases for example by mitigating restarts etc through failovers and redundancy, replicas should not be considered disaster recovery sources for the simple reason that they might be also corrupted or lost. Disaster recovery should mean that you have a path from being completely owned.

In practice your first consairn is backup. Either you use a cronjob to perform a series of dumps or volume snapshots by the database or through velero, you will be doing something similar to

```yaml
annotations:
  backup.velero.io/backup-volumes: backup
  post.hook.restore.velero.io/command: '["/bin/bash", "-c", "[ -f \"/scratch/backup.sql\" ] && PGPASSWORD=$POSTGRES_PASSWORD psql -U $POSTGRES_USER -h 127.0.0.1 -d $POSTGRES_DATABASE -f /scratch/backup.sql && rm -f /scratch/backup.sql;"]'
  pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "PGPASSWORD=$POSTGRES_PASSWORD pg_dump -U $POSTGRES_USER -d $POSTGRES_DATABASE -h 127.0.0.1 > /scratch/backup.sql && mkdir -p /bitnami/postgresql/backups && mv /scratch/backup.sql /bitnami/postgresql/backups"]'
  pre.hook.backup.velero.io/timeout: 15m
```

You can define these as complex as required using the RTO and RPO as definite guides. What you really need to pay special attention to is WHERE these are saved and WHY its safe. To be honest here the only real solution for this is to use some form of [vault lock](https://aws.amazon.com/blogs/aws/glacier-vault-lock/) with WORM capability. For example Amazon S3 supports WORM (Write Once Read Many) functionality through S3 Object Lock, which allows you to store objects in a way that they cannot be changed or deleted after they have been written. This feature is useful for regulatory compliance and data protection. In practise this means that even if the ROOT account of the cloud provider gets compromised the backups will be safe. You will need to also configure lifecycle policies to ensure that costs don't skyrocket. Even if you are running on-prem consider using something like [AWS PrivateLink](https://aws.amazon.com/privatelink/) to link your local network and store your backups on cloud.

Again keep in mind that here isolation is your best friend. If you restore a database you restore a database. This means that ALL your clients will be effected. Thus its very important to use devide and conquer here, meaning create backup and recovery plans that will allow you to perform a partial recovery for only the effected clients. This might mean that you dump all the database in the vault but allow your ops team to setup a partial restore proccess for the particular clients. Just have these defined and run recovery drills to verify you meet your RTO and RPO.

```mermaid
sequenceDiagram
    autonumber
    participant Ops as Ops Team
    participant BackupSys as Backup System
    participant WORM as WORM Storage (S3 Object Lock / Glacier Vault Lock)
    participant DB as Database
    participant Network as Secure Network (AWS PrivateLink)
    participant Client as Client Application

    Ops->>BackupSys: Initiate Backup Process
    BackupSys->>DB: Dump Database (Per Client)
    DB-->>BackupSys: Database Dump Created
    BackupSys->>Network: Transfer Backup (PrivateLink)
    Network-->>WORM: Store Backup (WORM Enabled)
    WORM-->>BackupSys: Backup Write Confirmed (Immutable Storage)

    Note right of WORM: Backup stored with vault lock (WORM).

    Ops->>WORM: Configure Lifecycle Policy
    WORM-->>Ops: Policy Enforced (Cost Control & Retention Management)

    Note over Ops, WORM: Define retention policies to prevent cost escalation.

    Client->>DB: Perform Transactions
    DB-->>Client: Confirmed (Live Data Updated)

    Note right of DB: Database serves live clients.

    alt Data Loss or Corruption Detected
        Ops->>WORM: Identify Affected Clients
        Ops->>BackupSys: Initiate Partial Restore for Clients
        BackupSys->>Network: Retrieve Affected Client Data
        Network-->>DB: Restore Client Data (Isolated)
        DB-->>Ops: Partial Restore Complete
    end

    Note over Ops: Regular recovery drills to ensure RTO & RPO compliance.


```

# Regarding scalability

This entire setup considers a single database that can be used for read and write. You can also create different instances and "schedule" your tenants to them to have even balances. But sooner or later you will probably need to leverage different paths for read and write. This means replication, and replication means eventual consistency. Ignoring for now the eventual consistency logic, the architecture for this is as follows:

1. Create a high-availability setup. You can use leader election or other strategies, but for simplicity let's assume that the PRIMARY is defined. The optimal approach here is to create read REPLICAS ensuring with anti-affinity that each read replica lives in a different AZ. This is especially important considering that block storage volumes are AZ-locked and can't easily migrate between zones.
2. Then you have the issue of query routing. Here again, there are two main approaches that work well, depending if you prefer to pay the development overhead or leverage a standard solution:
   1. Use a read and a write connection string, and have the application create different connections that produce a read and a read_write cursor to the database. Then the application explicitly selects what to use for each case.
   2. Use a service like pgPool that will act like a reverse proxy to the database and depending on its selection or update the query, route the query to the correct instance type. Personally, I like the latter approach as it gives this power to the infrastructure teams which are more aware of what runs where etc.

# Regarding observability

Sample Prometheus rule regarding a generic app database that can be used as a reference.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-postgresql
  labels:
    release: prometheus
    app.kubernetes.io/component: metrics
spec:
  groups:
    - name: app-postgresql
      rules:
        - alert: PostgresqlDown
          annotations:
            description: Postgresql instance is down VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql down (instance {{ $labels.instance }})
          expr: pg_up{namespace="backend", pod=~"app-postgresql-.*"} == 0
          for: 0m
          labels:
            severity: critical
        - alert: PrimaryPostgresqlRestarted
          annotations:
            description: Postgresql restarted VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql restarted (instance {{ $labels.instance }})
          expr: time() - process_start_time_seconds{namespace="backend", pod="app-postgresql-primary-0"} < 60
          for: 2m
          labels:
            severity: critical
        - alert: PostgresqlRestarted
          annotations:
            description: Postgresql restarted VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql restarted (instance {{ $labels.instance }})
          expr: time() - process_start_time_seconds{namespace="backend", pod=~"app-postgresql-.*"} < 60
          for: 0m
          labels:
            severity: warning
        - alert: PostgresqlRunningOutConnections
          annotations:
            description: Available VALUE  VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Number of available connections less than 10% (instance {{ $labels.instance }})
          expr: (((sum(pg_settings_max_connections{namespace="backend", pod=~"app-postgresql-.*"}) by (server) - sum(pg_settings_superuser_reserved_connections{namespace="backend", pod=~"app-postgresql-.*"}) by (server)) - sum(pg_stat_activity_count{namespace="backend", pod=~"app-postgresql-.*"}) by (server)) / sum(pg_settings_max_connections{namespace="backend", pod=~"app-postgresql-.*"}) by (server)) * 100 < 10
          for: 2m
          labels:
            component: database
            severity: warning
        - alert: PostgresqlExporterError
          annotations:
            description: Postgresql exporter is showing errors. A query may be buggy in query.yaml VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql exporter error (instance {{ $labels.instance }})
          expr: pg_exporter_last_scrape_error{namespace="backend", pod=~"app-postgresql-.*"} > 0
          for: 0m
          labels:
            severity: critical
        - alert: PostgresqlTableNotAutoVacuumed
          annotations:
            description: Table {{ $labels.relname }} has not been auto vacuumed for 10 days VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql table not auto vacuumed (instance {{ $labels.instance }})
          expr: (pg_stat_user_tables_last_autovacuum{namespace="backend", pod=~"app-postgresql-.*"} > 0) and (time() - pg_stat_user_tables_last_autovacuum{namespace="backend", pod=~"app-postgresql-.*"}) > 60 * 60 * 24 * 10
          for: 0m
          labels:
            severity: warning
        - alert: PostgresqlTableNotAutoAnalyzed
          annotations:
            description: Table {{ $labels.relname }} has not been auto analyzed for 10 days VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql table not auto analyzed (instance {{ $labels.instance }})
          expr: (pg_stat_user_tables_last_autoanalyze{namespace="backend", pod=~"app-postgresql-.*"} > 0) and (time() - pg_stat_user_tables_last_autoanalyze{namespace="backend", pod=~"app-postgresql-.*"}) > 24 * 60 * 60 * 10
          for: 0m
          labels:
            severity: warning
        - alert: PostgresqlTooManyConnections
          annotations:
            description: PostgreSQL instance has too many connections (> 80%). VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql too many connections (instance {{ $labels.instance }})
          expr: sum by (instance, job, server) (pg_stat_activity_count{namespace="backend",pod=~"app-postgresql-.*"}) > min by (instance, job, server) (pg_settings_max_connections{namespace="backend",pod=~"app-postgresql-.*"} * 0.8)
          for: 2m
          labels:
            severity: warning
        - alert: PostgresqlNotEnoughConnections
          annotations:
            description: PostgreSQL instance should have more connections (> 5) VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql not enough connections (instance {{ $labels.instance }})
          expr: sum by (datname) (pg_stat_activity_count{namespace="backend",pod=~"app-postgresql-.*", datname!~"template.*|postgres|readme_to_recover"}) < 5
          for: 2m
          labels:
            severity: warning
        - alert: PostgresqlDeadLocks
          annotations:
            description: PostgreSQL has dead-locks VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql dead locks (instance {{ $labels.instance }})
          expr: increase(pg_stat_database_deadlocks{namespace="backend",pod=~"app-postgresql-.*", datname!~"template.*|postgres|readme_to_recover"}[1m]) > 5
          for: 0m
          labels:
            severity: warning
        - alert: PostgresqlHighRollbackRate
          annotations:
            description: Ratio of transactions being aborted compared to committed is > 2  VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql high rollback rate (instance {{ $labels.instance }})
          expr: sum by (namespace,datname) ((rate(pg_stat_database_xact_rollback{namespace="backend",pod=~"app-postgresql-.*",datname!~"template.*|postgres|readme_to_recover",datid!="0"}[3m]))) / (((rate(pg_stat_database_xact_rollback{namespace="backend",pod=~"app-postgresql-.*",datname!~"template.*|postgres",datid!="0"}[3m])) + (rate(pg_stat_database_xact_commit{datname!~"template.*|postgres",datid!="0"}[3m])))) > 0.02
          for: 0m
          labels:
            severity: warning
        - alert: PostgresqlCommitRateLow
          annotations:
            description: Postgresql seems to be processing very few transactions VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql commit rate low (instance {{ $labels.instance }})
          expr: rate(pg_stat_database_xact_commit{namespace="backend",pod=~"app-postgresql-.*",datname!~"template.*|postgres|readme_to_recover"}[1m]) < 1
          for: 2m
          labels:
            severity: warning
        - alert: PostgresqlReplicationLag
          annotations:
            description: VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql replica is lagging (instance {{ $labels.instance }})
          expr: pg_replication_lag_seconds{namespace="backend",pod=~"app-postgresql-.*"} > 2
          for: 1m
          labels:
            severity: warning
        - alert: PostgresqlTooManyDeadTuples
          annotations:
            description: PostgreSQL dead tuples is too large VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql too many dead tuples (instance {{ $labels.instance }})
          expr: ((pg_stat_user_tables_n_dead_tup{namespace="backend",pod=~"app-postgresql-.*"} > 5)) / (pg_stat_user_tables_n_live_tup{namespace="backend",pod=~"app-postgresql-.*"} + pg_stat_user_tables_n_dead_tup{namespace="backend",pod=~"app-postgresql-.*"}) >= 0.1
          for: 2m
          labels:
            severity: warning
        - alert: PostgresqlTooManyLocksAcquired
          annotations:
            description: Too many locks were acquired on the database. If this alert happens frequently, we may need to increase the postgres setting max_locks_per_transaction. VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql too many locks acquired (instance {{ $labels.instance }})
          expr: ((sum (pg_locks_count{namespace="backend",pod=~"app-postgresql-.*"})) / (pg_settings_max_locks_per_transaction{namespace="backend",pod=~"app-postgresql-.*"} * pg_settings_max_connections{namespace="backend",pod=~"app-postgresql-.*"})) > 0.20
          for: 2m
          labels:
            severity: critical
        - alert: PostgresqlLatency
          annotations:
            description: Postgres is running slow. VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql is lagging over 1 second (instance {{ $labels.instance }})
          expr: pg_stat_activity_max_tx_duration{namespace="backend",pod=~"app-postgresql-.*",state="active"} > 1
          for: 2m
          labels:
            severity: warning
        - alert: PostgresqlCacheHitRate
          annotations:
            description: Postgres cache hit rate is very low. VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql is lagging over 1 second (instance {{ $labels.instance }})
          expr: 100 * (rate(pg_stat_database_blks_hit{namespace="backend",pod=~"app-postgresql-.*"}[2m]) / ((rate(pg_stat_database_blks_hit{namespace="backend",pod=~"app-postgresql-.*"}[2m]) + rate(pg_stat_database_blks_read{namespace="backend",pod=~"app-postgresql-.*"}[2m]))>0)) < 80
          for: 2m
          labels:
            severity: warning
        - alert: PostgresqlMemoryAvailable
          annotations:
            description: Postgres is using over 0.8 of available memory. VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql running out of available memory (instance {{ $labels.instance }})
          expr: sum by(pod,container)(container_memory_usage_bytes{namespace="backend",pod=~"app-postgresql-.*", container!="POD",container!=""}) / sum by(pod,container)(kube_pod_container_resource_limits{namespace="backend",pod=~"app-postgresql-.*",resource="memory"}) > 0.8
          for: 2m
          labels:
            severity: warning
        - alert: PostgresqlRequestedBufferCheckpoints
          annotations:
            description: PostgreSQL uses the buffer checkpoints to write the dirty buffers on disk, so it creates safe points for the Write Ahead Log (WAL). These checkpoints are scheduled periodically but also can be requested on-demand when the buffer runs out of space. A high number of requested checkpoints compared to the number of scheduled checkpoints can impact directly the performance of your PostgreSQL instance. To avoid this situation you could increase the database buffer size. VALUE = {{ $value }} LABELS = {{ $labels }}
            summary: Postgresql is over using write buffer (instance {{ $labels.instance }})
          expr: rate(pg_stat_bgwriter_checkpoints_req_total{namespace="backend",pod=~"app-postgresql-.*"}[5m]) / (rate(pg_stat_bgwriter_checkpoints_req_total{namespace="backend",pod=~"app-postgresql-.*"}[5m]) + rate(pg_stat_bgwriter_checkpoints_timed_total{namespace="backend",pod=~"app-postgresql-.*"}[5m])) * 100 > 0.8
          for: 2m
          labels:
            severity: warning
```

If you need some business-related logic to be exposed and alert on, then you can use the metrics exporter and create a custom metric, eg:

```yaml
pg_database:
  metrics:
  - name:
      description: Name of the database
      usage: LABEL
  - size_bytes:
      description: Size of the database in bytes
      usage: GAUGE
  query: SELECT pg_database.datname, pg_database_size(pg_database.datname) as bytes
    FROM pg_database;
```

and then create an alert based on the particular metric. Just try to keep in mind that these will be queries that will get regurarly executied every time prometheus scrapes the exporter. So maybe you don't need exact numbers and could use good estimations. Try and use `EXPLAIN ANALYZE` to optimize.

### How to make all these possible

These are the tools I leveraged:
