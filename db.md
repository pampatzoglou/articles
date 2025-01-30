### **Into (WIP)**

A journey into discovering how to handle state though databases in production environment with examples that I found useful. We will consider aspects like setup, isolation, disaster recovery with RTO and RPO, reliability, security, lifecycle management, costs and competing aspects.

### **1. Shared Database, Shared Schema**

* All tenants use the same database and the same set of tables.
* Each entry includes a ``tenant_id`` column to segregate data.
* Efficient use of resources is debatable because of the `WHERE tenant_id` will add overhead to each query.
* **Pros:**
  * Simplifies deployment and maintenance.
* **Cons:**
  * Requires strict tenant-aware data access controls. If you implement a ROW level ACL .... good luck.
  * Certain performance bottlenecks as the tenant count grows.
  * Extrimely bad If there is a DR schenario:
    * all users will be effected as you will need to rollback ALL the database to the last known good state.
    * botched operations will be global.
    * global recovery increases MTTR.

> A bad idea here to shards tenants would be to use table prefixes. **DONT**.

### 2. **Shared Database, Separate Schema**

* Each tenant has its own schema within a shared database. Table names etc within the schema should be consistent to simplify development.
* Schema-based isolation improves security and performance. From security perspective this isolation is considered weak because you need to protect from:
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
  * Still need to have strict database logs and alerts to detect unauthorized access because all users will be working on the same proccess.
  * Complicated backups because each schema should have a dedicated backup proccess.
  * Can't use volume snapshots for backups. They will rever the global state meaning you effect all tenants.

### 3. **Separate Databases per Tenant, Shared database instance**

* Each tenant has its own dedicated database but all tenants use the same CPUs.
* Ensures complete data isolation.
* **Pros:**
  * All benefits for Shared Database, Separate Schema setup plus:
  * Strong ACL offer improved security.
  * Eases tenant-specific backups and compliance handling.
  * Simplest and fastest DR.
  * DR is not global.
* **Cons:**
  * Migration and schema updates still complex.
  * Higher infrastructure and management overhead. Will discuss solution later in this doc.
  * Efficient scaling needs more thought.

### 4. Separate dabases instance per tenant

* Each tenant is running on their own database, probably in their dedicated namespace. Or perhaps on a different region etc. At this level of isolation it doesnt matter.
* Even stronger isolation because you don't rely only on db level ACL but also network policies/firewall rules.
* In essence its a special case of the 3rd option with:
* **Pros:**

  * Even stronger isolation
* **Cons:**

  * Capacity planning and optimisations overhead will be significant.
  * To benefit from this level you need to ensure that the undelaying infrastructure setup is extrimely secure so your procceses are not exposed to other people running on the same cloud infra eg
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

### How to make all thesew possible

These are the tools I leveraged:

#### helm / argo hooks

Use two different jobs to handle the distinct aspects of the application lifecycle. The first Job runs first only on installation and its single responsibility is to create the database with the required parameters. In fact it will run even before installation as we use the `pre-install` hook, which means before any application spesific resources are created.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "chart.fullname" . }}-db-create-{{ randAlphaNum 5 | lower }}
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "0"
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
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": before-hook-creation
    "argocd.argoproj.io/hook": "PreSync"
    "argocd.argoproj.io/hook-delete-policy": "BeforeHookCreation"
    "argocd.argoproj.io/job-cleanup": "keep"
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-migrate
```

**note**: The db create entity requires superuser priviledges to actually create the database. Thus its better to not even trust the cluster to save a secret which contains this level of credentials. A better approach is to have a binary that can directly access the vault and directly get the required credentials. The `db migrate` job should apply the same principle but there will be problems with local development. As a compromise two different secrets with different RBAC should be used to facilitate production security and development ease. Something like:

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

#### Zookeeper

Different databases, maybe different ingresses, different shit. Unless the architecture allows for simple configuration managment I use zookeeper to keep all these in check. Each deployment's variables, are saved in zookeeper with groups etc. To offer a central place for handling configuration. But this will need to be handled by application logic as there is no `zookeeper operator` to create dynamic configs. So in a nutshel if the devs can take it, use zookeeper, if not, handle these through the next option, the vault.

#### Hashicorp vault

I consider vault the root user of all databases. Its given access to all of them and with secret engines, eg [postgres](https://developer.hashicorp.com/vault/docs/secrets/databases/postgresql) it can generate [ephimeral credentials](https://developer.hashicorp.com/vault/docs/secrets/databases/postgresql).

These credentials can be mapped to IAM roles, or use [external secrets operator](https://external-secrets.io/latest/introduction/overview/) or vault secrets operator. Which ever approach in the end, a short lived username/password is generated that access the single database or schema with the required permissions. Keep in mind that these should be different for the proccess that handles schema (which needs to have elevated access eg to create table) and for the normal app/role that will be allowed to only read/write/update but never delete or drop. The beauty of this is that from the developers perspective its the same secret (though different values) that the application uses. So they will not need to bother with the overhead of this logic in the application code. The downside of this approach is that when values change, it will require that the pod gets restated which may cascade and have issues with PDB.

An example generate the aforemention secrets using the vault and external secrets operator is:

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

Some more [examples](https://dev.to/breda/dynamic-postgresql-credentials-using-hashicorp-vault-with-php-symfony-go-examples-4imj)

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

The main problem is that by nature of vault etc, not all of these can be automated. It can be scripted but a person who actually has access to the vault will need to be involved to run the required configurations. Unless you create a custom opperator that has admin vault access and performs the required actions to setup the particular engine and paths etc. A better approach is to use the vault agent

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

### Regarding scalability...

This entire setup considers a single database that can be used for read and write. You can also create different instances and "schedule" your tenants to them to have even balances. But sooner or later you will probably need to leverage different paths for read and write. This means replication, and replication means eventual consistency. Ignoring for now the eventual consistency logic, the architecture for this is as follows:

1. Create a high availability setup. You can use leader election or other strategies, but for simplicity lets assume that the PRIMARY replica is defined. The optimal approach here is to create read REPLICAS ensuring with anti affinity that each read replica lives in a different AZ. This is especially important considering that block storages are AZ locked and cant easily migrate between zones.
2. Then you have the issue of query routing. Here again there are two main approaches that work well, depending if you prefer to pay the development overhead or leverage a standard solution:
   1. Use a read and a write connection string, and have the application create different connections which produce a read and a read_write cursor to the database. Then the application explisitly selects what do use for each case.
   2. Use a service like pgPool that will act like a reverse proxy to the database and depending if its a select or update query, route the query to the correct instance type. Personally I like the latter approach as is gives this power to the infrastucture teams which are more aware of what runs where etc.

To be continued....
