# redis-backup-webinar-may-23


## Deploy Redis Cluster 
```
apiVersion: kubedb.com/v1alpha2
kind: Redis
metadata:
  name: redis-cluster
  namespace: demo
spec:
  disableAuth: true
  version: 6.2.5
  mode: Cluster
  cluster:
    master: 3
    replicas: 1 
  storageType: Durable
  storage:
    resources:
      requests:
        storage: "1Gi"
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
  terminationPolicy: WipeOut
```


### Insert data

```
$ kubectl exec -it -n demo redis-cluster-shard0-0 -- bash
$ redis-cli -c set apps code
$ redis-cli -c set name shaad
$ redis-cli -c set hello world
```

## Prepare backup

### Create cloud secret

```
$ echo -n 'your-password' > RESTIC_PASSWORD
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ cat downloaded-sa-json.key > GOOGLE_SERVICE_ACCOUNT_JSON_KEY
```

```
kubectl create secret generic -n demo gcs-secret \
          --from-file=./RESTIC_PASSWORD \
          --from-file=./GOOGLE_PROJECT_ID \
          --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
```

### Create repository
```
apiVersion: stash.appscode.com/v1alpha1
kind: Repository
metadata:
  name: gcs-repo
  namespace: demo
spec:
  backend:
    gcs:
      bucket: stash-testing
      prefix: /demo/redis/redis-cluster
    storageSecretName: gcs-secret
```

### Create backp configuration

```
apiVersion: stash.appscode.com/v1beta1
kind: BackupConfiguration
metadata:
  name: redis-backup
  namespace: demo
spec:
  schedule: "*/5 * * * *"
  repository:
    name: gcs-repo
  target:
    ref:
      apiVersion: appcatalog.appscode.com/v1alpha1
      kind: AppBinding
      name: redis-cluster
  retentionPolicy:
    name: keep-last-5
    keepLast: 5
    prune: true
```

Trigger instant backupsession 
```
kubectl stash trigger -n demo redis-backup
```

## Restore Session
```
apiVersion: stash.appscode.com/v1beta1
kind: RestoreSession
metadata:
  name: redis-restore
  namespace: demo
spec:
  repository:
    name: gcs-repo
  target:
    ref:
      apiVersion: appcatalog.appscode.com/v1alpha1
      kind: AppBinding
      name: redis-cluster
  rules:
  - snapshots: [latest]

```