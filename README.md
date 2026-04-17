# Archive and push Victoria logs to S3 cold storage

This cronjob does the following
### Init Container
- mounts both backup and victoria logs volume 
- runs daily at 1am
- creates a snapshot of yesterdays logs(partition)
- creates a tar archive of the snapshot
- deletes the created snapshot from the victoria logs volume

### Main Container
- uses `rclone` command to push the created archive to s3
- deletes the created archive from the volume.

## Steps

Firstly we need to create a storage for cronjob to store archive files

```sh
kubectl apply -f backup-pvc.yml
```

Create a secret to store s3 credentials to be used by `rclone`.

```sh
kubectl create secret generic rclone-s3-config --from-file rclone.conf -n logging
```

Before creating cronjob we need to update `VLOGS_URL` in the script [init container] with the a proper url.

Create the cronjob.

```sh
kubectl apply -f cronjob.yml
```

Trigger the job now

```sh
kubectl create job --from=cronjob/vlogs-daily-backup vlogs-backup-test -n logging
```

## Restore Process

We can download the snapshot archive from s3 and attach the parition for querying.
http://server-url:9428/internal/partition/attach?name=YYYYMMDD - attaches the partition directory with the given name YYYYMMDD to VictoriaLogs, so it becomes visible for querying and can be used for data ingestion. The directory must be placed inside <-storageDataPath>/partitions and it must contain valid data for the given YYYYMMDD day.

## Terminology

### What Is a Snapshot?
A snapshot is a read-only (or read-write) point-in-time image of a filesystem, volume, or dataset. Unlike traditional backups, snapshots use copy-on-write (CoW) technology: they initially consume minimal disk space by referencing the original data. Only when data on the original volume changes does the snapshot store the old version of the modified blocks, preserving the original state.

## References
- https://docs.victoriametrics.com/victorialogs/#backup-and-restore
- https://docs.victoriametrics.com/victorialogs/#partitions-lifecycle