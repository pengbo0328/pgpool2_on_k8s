# Run Pgpool-II on Kubernetes

This documentation explains how to run [Pgpool-II](https://pgpool.net "Pgpool-II") and PostgreSQL Streaming Replication with [KubeDB](https://kubedb.com/ "KubeDB") on Kubernetes.

## Introduction

In a database cluster, replicas can't be created as easily as web servers, because you must consider
the difference between Primary and Standby. PostgreSQL operators simplify the processes of deploying
and managing a PostgreSQL cluster on Kubernetes. In this documentation, we use KubeDB to deploy and
manage a PostgreSQL cluster.

And on kubernetes Pgpool-II's health check, automatic failover, Watchdog and online recovery feature aren't requried. You need to only enable load balancing and connection pooling.

## Requirements

Before you start the install and configuration processes, please check the following prerequisites.
- Make sure you have a Kubernetes cluster, and the kubectl is installed.
- Kebernetes 1.15 or older is required.

## Install KubeDB Operator

We use a seperate namespace to install KubeDB.

```
# kubectl create namespace demo
# curl -fsSL https://raw.githubusercontent.com/kubedb/installer/v0.13.0-rc.0/deploy/kubedb.sh | bash -s -- --namespace=demo
```

After installing, a running KubeDB-operator pod is created.

```
# kubectl get pod -n demo
NAME                              READY   STATUS    RESTARTS   AGE
kubedb-operator-5565fbdb8-22ks9   1/1     Running   1          5m32s
```

## Install PostgreSQL SR with Hot Standby

### Create secret

By default, superuser name is postgres and password is randomly generated.
If you want to use a custom password, please create the secret manually.
The data specified in a secret need to be encoded using base64.
Here we set `postgres` as `postgres` user's password.

```
# echo -n 'postgres' | base64
cG9zdGdyZXM=
# kubectl apply -f https://raw.githubusercontent.com/pgpool/pgpool2_on_k8s/master/hot-postgres-auth.yaml
```

### Create PostgreSQL cluster with streaming replication

We use KubeDB to create a PostgreSQL cluster with Monitoring Enabled.
Below is an example of Postgres object which creates a PostgreSQL cluster (1 Primary and 2 Standby)
with Monitoring Enabled.

```
apiVersion: kubedb.com/v1alpha1
kind: Postgres
metadata:
  name: hot-postgres
spec:
  version: "11.2"
  replicas: 3
  standbyMode: Hot
  databaseSecret:
    secretName: hot-postgres-auth
  storageType: Durable
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  monitor:
    agent: prometheus.io/builtin
```

- `spec.replicas: 3` specifies that we create three PostgreSQL pods
- `spec.standbyMode: Hot` specifies that one server is Primary server and two others are Standby servers
- `spec.monitor.agent: prometheus.io/builtin` enables build-in monitoring using Prometheus

```
# kubectl apply -f https://raw.githubusercontent.com/pgpool/pgpool2_on_k8s/master/hot-postgres.yaml --namespace=demo
```

After applying the YAML file above you can see three pods are created.
`hot-postgres-0` is Primary server and `hot-postgres-1` and `hot-postgres-2` are Standby servers.

```
# kubectl get pod -n demo --selector="kubedb.com/name=hot-postgres" --show-labels
NAME             READY   STATUS    RESTARTS   AGE     LABELS
hot-postgres-0   2/2     Running   0          7m35s   controller-revision-hash=hot-postgres-69bd777947,kubedb.com/kind=Postgres,kubedb.com/name=hot-postgres,kubedb.com/role=primary,statefulset.kubernetes.io/pod-name=hot-postgres-0
hot-postgres-1   2/2     Running   0          6m9s    controller-revision-hash=hot-postgres-69bd777947,kubedb.com/kind=Postgres,kubedb.com/name=hot-postgres,kubedb.com/role=replica,statefulset.kubernetes.io/pod-name=hot-postgres-1
hot-postgres-2   2/2     Running   0          4m48s   controller-revision-hash=hot-postgres-69bd777947,kubedb.com/kind=Postgres,kubedb.com/name=hot-postgres,kubedb.com/role=replica,statefulset.kubernetes.io/pod-name=hot-postgres-2
```

And two `Service` are created.
`hot-postgres` service is mapped to Primary server and
`hot-postgres-replicas` service is mapped to Standby servers.

```
# kubectl get svc -n demo
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
hot-postgres            ClusterIP   10.106.170.103   <none>        5432/TCP    9m9s
hot-postgres-replicas   ClusterIP   10.107.182.173   <none>        5432/TCP    9m9s
hot-postgres-stats      ClusterIP   10.96.80.254     <none>        56790/TCP   4m10s
kubedb                  ClusterIP   None             <none>        <none>      9m9s
kubedb-operator         ClusterIP   10.110.0.111     <none>        443/TCP     14m
```

## Deploy Pgpool-II

Next, let's deploy Pgpool-II pod that contains a Pgpool-II container and a [Pgpool-II Exporter](https://github.com/pgpool/pgpool2_exporter "Pgpool-II Exporter") container.

Environment variables starting with `PGPOOL_PARAMS_` can be converted to Pgpool-II's configuration parameters
and these environment variables can override the default configuration.

For example, here we set Primary and Standby `Service` name to environment variables.

```
env:
- name: PGPOOL_PARAMS_BACKEND_HOSTNAME0
  value: "hot-postgres"
- name: PGPOOL_PARAMS_BACKEND_HOSTNAME1
  value: "hot-postgres-replicas"
```

The environment variables above will be convert to the following configurations.

```
backend_hostname0='hot-postgres'
backend_hostname1='hot-postgres-replicas'
```

```
# kubectl apply -f https://raw.githubusercontent.com/pgpool/pgpool2_on_k8s/master/pgpool_deploy.yaml --namespace=demo
```

Alternatively, if you want to modify more Pgpool-II parameters, you can configure Pgpool-II using `ConfigMap`.

```
# kubectl apply -f https://raw.githubusercontent.com/pgpool/pgpool2_on_k8s/master/pgpool_configmap.yaml --namespace=demo
# kubectl apply -f https://raw.githubusercontent.com/pgpool/pgpool2_on_k8s/master/pgpool_deploy_with_mount_configmap.yaml --namespace=demo
```

```
# kubectl get pod -n demo
NAME                              READY   STATUS    RESTARTS   AGE
hot-postgres-0                    2/2     Running   0          9m4s
hot-postgres-1                    2/2     Running   0          7m38s
hot-postgres-2                    2/2     Running   0          6m17s
kubedb-operator-5565fbdb8-22ks9   1/1     Running   1          13m
pgpool-55cfbcb9cb-8fm6f           2/2     Running   0          12s

[root@ser1 kube]# kubectl get svc -n demo
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
hot-postgres            ClusterIP   10.106.170.103   <none>        5432/TCP    9m9s
hot-postgres-replicas   ClusterIP   10.107.182.173   <none>        5432/TCP    9m9s
hot-postgres-stats      ClusterIP   10.96.80.254     <none>        56790/TCP   4m10s
kubedb                  ClusterIP   None             <none>        <none>      9m9s
kubedb-operator         ClusterIP   10.110.0.111     <none>        443/TCP     14m
pgpool                  ClusterIP   10.97.99.254     <none>        9999/TCP    17s
pgpool-stats            ClusterIP   10.98.225.77     <none>        9719/TCP    17s
```

## Deploy prometheus server

Configure Prometheus Server using `ConfigMap`.

```
# kubectl create namespace monitoring
# kubectl apply -f https://raw.githubusercontent.com/pgpool/pgpool2_on_k8s/master/prometheus_configmap.yaml --namespace=monitoring
```

Deploy prometheus server.

```
# kubectl apply -f https://raw.githubusercontent.com/pgpool/pgpool2_on_k8s/master/prometheus.yaml --namespace=monitoring

# kubectl get pod -n monitoring
NAME                          READY   STATUS    RESTARTS   AGE
prometheus-69bf7dc56f-h5xvh   1/1     Running   0          47h

# kubectl get svc -n monitoring
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
prometheus   ClusterIP   10.99.221.224   <none>        9090/TCP   47h
```

Forward 9090 port of `prometheus-69bf7dc56f-h5xvh` pod.

```
# kubectl port-forward -n monitoring prometheus-69bf7dc56f-h5xvh 9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

Now, you can access http://localhost:9090 in your browser.
