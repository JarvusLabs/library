# Restic Snapshots

## Install restic binary

Install the latest `restic` binary from your package manage of choice or via Jarvus' Chef Habitat package:

```bash
hab pkg install --binlink jarvus/restic
```

## Provision bucket

Create a bucket and obtain access credentials at a low-cost cloud object storage host:

=== "Linode"
    1. [Add an Object Storage bucket](https://cloud.linode.com/object-storage/buckets)
    2. [Create an Access key](https://cloud.linode.com/object-storage/access-keys)

=== "Backblaze"
    1. [Provision B2 bucket](https://secure.backblaze.com/b2_buckets.htm)
        - Select *Private*
        - Enable *Object Lock*
    2. [Provision B2 app key](https://secure.backblaze.com/app_keys.htm)
        - Use same name as bucket
        - Only allow access to created bucket
        - Select *Read and Write* access
        - Do not allow listing all bucket names
        - Do not set file name prefix or duration

## Build restic environment

Create a secure file to store needed environment variables for the `restic` client to read and write to the encrypted repository bucket:

=== "Linode"
    ```bash
    RESTIC_REPOSITORY=s3:us-east-1.linodeobjects.com/restic-myhost
    RESTIC_PASSWORD=

    # Access Key:
    AWS_ACCESS_KEY_ID=

    # Secret Key:
    AWS_SECRET_ACCESS_KEY=
    ```

=== "Backblaze"
    ```bash
    RESTIC_REPOSITORY=b2:restic-myhost
    RESTIC_PASSWORD=

    # keyID:
    B2_ACCOUNT_ID=

    # applicationKey:
    B2_ACCOUNT_KEY=
    ```

1. Create `~/.restic/myhost.env` from above template
2. Tailor `RESTIC_REPOSITORY` to created bucket
3. Generate `RESTIC_PASSWORD` and save to credential vault
4. Fill storage credentials
5. Secure configuration:

    ```bash
    sudo chmod 600 ~/.restic/myhost.env
    ```

## Initialize repository

Load the environment into your current shell to run Restic's one-time `init` command to verify connection details and set up the encrypted repository structure within the bucket:

```bash
set -a; ~/.restic/myhost.env; set +a
restic init
```

## Create Kubernetes Secret

Connect to the target cluster and create a new secret from the above `.env` file:

```bash
kubectl -n myhost \
    create secret generic restic \
    --from-env-file=$HOME/.restic/myhost.env
```

## Create Kubernetes CronJob

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-cron
  namespace: myapp

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-cron
  namespace: myapp
rules:
  - apiGroups:
      - ''
    resources:
      - pods
    verbs:
      - get
      - watch
      - list
  - apiGroups:
      - ''
    resources:
      - pods/exec
    verbs:
      - create

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-cron
  namespace: myapp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: myapp-cron
subjects:
  - kind: ServiceAccount
    name: myapp-cron

---

apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: myapp-hourly
  namespace: myapp
spec:
  concurrencyPolicy: Forbid
  schedule: 40 * * * *
  startingDeadlineSeconds: 86400
  jobTemplate:
    spec:
      activeDeadlineSeconds: 7200
      template:
        spec:
          serviceAccountName: myapp-cron
          restartPolicy: Never
          containers:
            - name: kubectl
              image: lachlanevenson/k8s-kubectl
              envFrom:
                - secretRef:
                    name: restic
              args:
              command:
                - /bin/sh
                - '-c'
                - >
                  # resolve pod name for running instance (there should just be
                  one)

                  pod_name=$(kubectl get pod \
                      -l app.kubernetes.io/instance=myapp \
                      --field-selector=status.phase==Running \
                      -o jsonpath='{.items[0].metadata.name}'
                  )


                  # snapshot mysql database and site data to remote restic repository
                  kubectl exec "${pod_name}" -- bash -c "

                    # configure restic repository
                    export B2_ACCOUNT_ID='${B2_ACCOUNT_ID}'
                    export B2_ACCOUNT_KEY='${B2_ACCOUNT_KEY}'
                    export RESTIC_REPOSITORY='${RESTIC_REPOSITORY}'
                    export RESTIC_PASSWORD='${RESTIC_PASSWORD}'

                    # install CLI dependencies
                    hab pkg install jarvus/restic core/gzip core/curl

                    # get current database name
                    database_name=\"\$(hab pkg exec myorigin/myapp-composite mysql -srNe 'SELECT SCHEMA()')\"

                    # snapshot current database
                    echo \"Snapshotting database: \${database_name}\"
                    hab pkg exec myorigin/myapp-composite \
                      mysqldump \
                        --default-character-set=utf8 \
                        --force \
                        --single-transaction \
                        --quick \
                        --compact \
                        --extended-insert \
                        --order-by-primary \
                        --ignore-table=\"\${database_name}.sessions\" \
                        \"\${database_name}\" \
                      | hab pkg exec core/gzip gzip --rsyncable \
                      | hab pkg exec jarvus/restic restic backup \
                        --host 'myapp' \
                        --stdin \
                        --stdin-filename database.sql.gz

                    # snapshot data
                    echo 'Snapshoting site data'
                    hab pkg exec jarvus/restic restic backup \
                      /hab/svc/myapp/data \
                      --host 'myapp' \
                      --exclude='*.log' \
                      --exclude='/hab/svc/myapp/data/media/*x*/**'

                    # prune aged snapshots
                    echo 'Pruning snapshots'
                    hab pkg exec jarvus/restic restic forget \
                      --host 'myapp' \
                      --keep-last 3 \
                      --keep-daily 7 \
                      --keep-weekly 52
                  "
```
