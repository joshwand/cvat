---
title: 'Backup guide'
linkTitle: 'Backup guide'
weight: 11
description: 'Instructions on how to backup CVAT data with Docker.'
---

<!--lint disable heading-style-->

## About CVAT data volumes

Docker volumes are used to store all CVAT data:

- `cvat_db`: PostgreSQL database files, used to store information about users, tasks, projects, annotations, etc.
  Mounted into `cvat_db` container by `/var/lib/postgresql/data` path.

- `cvat_data`: used to store uploaded and prepared media data.
  Mounted into `cvat` container by `/home/django/data` path.

- `cvat_keys`: used to store user ssh keys needed for [synchronization with a remote Git repository](/docs/manual/advanced/task-synchronization/).
  Mounted into `cvat` container by `/home/django/keys` path.

- `cvat_logs`: used to store logs of CVAT backend processes managed by supevisord.
  Mounted into `cvat` container by `/home/django/logs` path.

- `cvat_events`: this is an optional volume that is used only when [Analytics component](/docs/administration/advanced/analytics/)
  is enabled and is used to store Elasticsearch database files.
  Mounted into `cvat_elasticsearch` container by `/usr/share/elasticsearch/data` path.

## How to backup all CVAT data

All CVAT containers should be stopped before backup:

```bash
docker-compose stop
```

Please don't forget to include all the compose config files that were used in the docker-compose command
using the `-f` parameter.

Backup data:

```bash
mkdir backup
docker run --rm --name temp_backup --volumes-from cvat_db -v $(pwd)/backup:/backup ubuntu tar -cjvf /backup/cvat_db.tar.bz2 /var/lib/postgresql/data
docker run --rm --name temp_backup --volumes-from cvat_server -v $(pwd)/backup:/backup ubuntu tar -cjvf /backup/cvat_data.tar.bz2 /home/django/data
# [optional]
docker run --rm --name temp_backup --volumes-from cvat_elasticsearch -v $(pwd)/backup:/backup ubuntu tar -cjvf /backup/cvat_events.tar.bz2 /usr/share/elasticsearch/data
```

Make sure the backup archives have been created, the output of `ls backup` command should look like this:

```bash
ls backup
cvat_data.tar.bz2  cvat_db.tar.bz2  cvat_events.tar.bz2
```

## How to restore CVAT from backup

**Warning: use exactly the same CVAT version to restore DB. Otherwise
it will not work because between CVAT releases the layout of DB can be
changed. You always can upgrade CVAT later. It will take care to migrate
your data properly internally.**

Note: CVAT containers must exist (if no, please follow the [installation guide](/docs/administration/basics/installation/#quick-installation-guide)).
Stop all CVAT containers:

```bash
docker-compose stop
```

Restore data:

```bash
cd <path_to_backup_folder>
docker run --rm --name temp_backup --volumes-from cvat_db -v $(pwd):/backup ubuntu bash -c "cd /var/lib/postgresql/data && tar -xvf /backup/cvat_db.tar.bz2 --strip 4"
docker run --rm --name temp_backup --volumes-from cvat_server -v $(pwd):/backup ubuntu bash -c "cd /home/django/data && tar -xvf /backup/cvat_data.tar.bz2 --strip 3"
# [optional]
docker run --rm --name temp_backup --volumes-from cvat_elasticsearch -v $(pwd):/backup ubuntu bash -c "cd /usr/share/elasticsearch/data && tar -xvf /backup/cvat_events.tar.bz2 --strip 4"
```

After that run CVAT as usual:

```bash
docker-compose up -d
```

## Additional resources

[Docker guide about volume backups](https://docs.docker.com/storage/volumes/#backup-restore-or-migrate-data-volumes)
