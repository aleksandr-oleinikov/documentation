---
description: HA AidboxDB installation with Crunchy
---

# HA AidboxDB

High availability for PostgreSQL is complex because it requires multiple components to work seamlessly, can be time-consuming to set up and configure manually, and ongoing maintenance can be challenging.&#x20;

Using ready solutions like the Crunchy operator for Kubernetes simplifies the process and improves reliability. Crunchy and similar operators provide a tested and production-ready infrastructure that integrates well with PostgreSQL, as well as features like automatic failover, backups, restores, and upgrades, which can be complex to implement manually. Overall, using a ready solution like Crunchy can reduce complexity and free up time and resources to focus on other aspects of your application.

### Crunchy Operator

The [Crunchy Operator](https://access.crunchydata.com/documentation/postgres-operator/5.3.1/quickstart/) is an open-source Kubernetes operator that automates the management of PostgreSQL clusters. It provides a simple way to deploy, manage, and operate PostgreSQL clusters in a Kubernetes environment, making it easier to run and scale PostgreSQL workloads.

One of the key benefits of using the Crunchy operator is that it allows for high availability and fault tolerance for your PostgreSQL database. When running a PostgreSQL cluster with the Crunchy operator, you can specify the number of replicas to create, which helps ensure that your database is always available in case of a failure.

Here's how high availability works in terms of the Crunchy operator:

* The Crunchy operator deploys a primary PostgreSQL instance and one or more replicas.
* The primary instance is responsible for accepting read and write requests and replicating changes to the replicas.
* If the primary instance fails, one of the replicas is promoted to become the new primary instance.
* The Crunchy operator automatically reconfigures the remaining replicas to replicate from the new primary instance.
* This ensures that the cluster remains available even if one or more instances fail.

In addition to high availability, the Crunchy operator also provides other features such as backups and restores, scaling, rolling upgrades, and custom configurations using PostgreSQL custom resource definitions (CRDs).

### Install Crunchy

We recommend following official Crunchy [Quickstart](https://access.crunchydata.com/documentation/postgres-operator/v5/quickstart/) for how to install and get up and running with PGO. Here are some instructions to get Postgres up and running on Kubernetes:

1. [Fork the Postgres Operator examples repository](https://github.com/CrunchyData/postgres-operator-examples/fork) and clone it to your host machine.

```bash
YOUR_GITHUB_UN="<your GitHub username>"
git clone --depth 1 "git@github.com:${YOUR_GITHUB_UN}/postgres-operator-examples.git"
cd postgres-operator-examples
```

2. Install PGO using `kustomize`

```bash
kubectl apply -k kustomize/install/namespace
kubectl apply --server-side -k kustomize/install/default
```

3. Werify PGO

```bash
$ kubectl get pods -n postgres-operator
NAME                           READY   STATUS    RESTARTS   AGE
pgo-7b5d478777-7g6kc           1/1     Running   0          51m
pgo-upgrade-5b576ccfb5-m5qdc   1/1     Running   0          51m
```

### Create cluster

For creating new PostgreSQL cluster using PGO you should create CRD `PostgresCluster.` More detailed information about creating a PGO cluster you can found in [official documentation](https://access.crunchydata.com/documentation/postgres-operator/5.3.1/tutorial/create-cluster/).

1. Create `aidboxdb.yml` file with the following content

<pre class="language-yaml" data-title="aidboxdb.yml"><code class="lang-yaml"><strong>apiVersion: postgres-operator.crunchydata.com/v1beta1
</strong>kind: PostgresCluster
metadata:
  name: aidboxdb
  namespace: aidboxdb-db
spec:
  image: healthsamurai/aidboxdb:15.2.0-crunchy
  postgresVersion: 15
  port: 5432
  instances:
    - name: aidboxdb
      replicas: 2
      dataVolumeClaimSpec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
  backups:
    pgbackrest:
      image: registry.developers.crunchydata.com/crunchydata/crunchy-pgbackrest:ubi8-2.41-4
      manual:
        repoName: repo1
        options:
         - --type=full
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 10Gi
  users:
    - databases:
        - aidbox
      name: aidbox
      options: "SUPERUSER"
  patroni:
    dynamicConfiguration:
      postgresql:
        parameters:
          listen_addresses : '*'
          shared_preload_libraries : 'pg_stat_statements'
          shared_buffers : '1GB'

</code></pre>

Important notes

* `image: healthsamurai/aidboxdb:15.2.0-crunchy`  - we recommend use our aidboxdb image build that is preconfigured for using in PGO
* `replicas: 2`  - in this configuration we install 1 master and 1 replica
* `backup options` - in this sample we use local PVC for storing backups. For configuring cloud storages like S3 or GCS you can [follow this instructions](https://access.crunchydata.com/documentation/postgres-operator/5.3.1/tutorial/backups/)

2. Create namespace and apply aidboxdb.yml resource

```bash
$ kubectl create ns aidboxdb-db
namespace/aidboxdb-db created                                                                                                      ⎈ kind-kind 11:21:57
$ kubectl apply -f aidboxdb.yaml
postgrescluster.postgres-operator.crunchydata.com/aidboxdb created
```

3. Werify postgresql cluster

```bash
$ kubectl get pods -n aidboxdb-db
NAME                         READY   STATUS      RESTARTS   AGE
aidboxdb-aidboxdb-p2tm-0     4/4     Running     0          12m
aidboxdb-aidboxdb-tc58-0     4/4     Running     0          12m
aidboxdb-backup-qvk7-q7qmv   0/1     Completed   0          11m
aidboxdb-repo-host-0         2/2     Running     0          12m
```

### Connect to the cluster&#x20;

* get secrets&#x20;
* get hosts&#x20;
* configure aidbox

### Backup a cluster



### Restore PITR


