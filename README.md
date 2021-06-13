# Ceph Object Storage

Deploy ceph version 16.2.4 & Container version v6.0.3-stable-6.0-pacific-centos-8

## Setup

[more info](https://docs.ceph.com/en/pacific/mgr/dashboard/)

## Quick Start

1. create secrets

   ```bash
   mkdir secrets
   echo DASHBOARD_PASSWORD > secrets/dashboard_password
   # Create empty files
   touch secrets/rgw_access_key
   touch secrets/rgw_secret_key
   ```

1. create bridge network

   ``` bash
   docker network create --driver bridge ceph
   ```

1. up monitor

   ``` bash
   docker-compose up -d mon1
   ```

1. configure ceph

   Limit bluestore block size to 100Mi, else will use default `100Gi` for docker volume.

   1. Option 1: Change configuration database

      ```bash
      docker exec -it ceph-mon ceph
      ceph> config set mon auth_allow_insecure_global_id_reclaim false
      ceph> config set global osd_pool_default_size 1
      ceph> config set global osd_pool_default_min_size 0
      ceph> config set global bluestore_block_size 100Mi
      ceph> exit
      ```

   1. Option 2: Edit `/etc/ceph/ceph.conf`

      ``` bash
      docker exec -it ceph-mon vi /etc/ceph/ceph.conf
      ```

      ``` conf
      osd pool default size = 1
      osd pool default min size = 0
      max open files = 655350
      bluestore_block_size = 100Mi
      ```

      Restart monitor.

      ```sh
      docker-compose restart mon1
      ```

1. start manager

   ``` bash
   docker-compose up -d mgr
   ```

1. up Object Storage Daemon

   ``` bash
   docker-compose exec mon1 ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
   docker-compose up -d osd1
   # Verify OSD is running
   docker-compose exec mon1 ceph status
   # See storage details
   docker-compose exec mon1 ceph df
   ```

1. up RADOS Gateway

   ``` bash
   docker-compose exec mon1 ceph auth get client.bootstrap-rgw -o /var/lib/ceph/bootstrap-rgw/ceph.keyring
   docker-compose up -d rgw1
   ```

1. enable `mgr` dashboard module

   ``` bash
   docker-compose exec mon1 ceph mgr module enable dashboard

   # Create user `admin` with role `administrator`
   docker-compose exec mon1 ceph dashboard ac-user-create admin -i /run/secrets/dashboard_password administrator

   # Create SSL
   docker-compose exec mon1 ceph dashboard create-self-signed-cert
   # Optional: Disable SSL
   # docker-compose exec mon1 ceph config set mgr mgr/dashboard/ssl false

   docker-compose exec mon1 ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
   docker-compose exec mon1 ceph config set mgr mgr/dashboard/server_port 8443

   # Restart dashboard module
   docker-compose exec mon1 ceph mgr module disable dashboard
   docker-compose exec mon1 ceph mgr module enable dashboard

   # Visit dashboard https://localhost:8443
   ```

1. verify mgr services

   ``` bash
   docker-compose exec mon1 ceph mgr services
   ```

1. create rados gateway user

   ``` bash
   docker-compose exec mon1 radosgw-admin user create --uid=rgw --display-name="RGW User" --system

   # Update ./secrets/{access_key, secret_key} with generated access key and secret
   ```

1. bind rados gatway user to dashboard

   ``` bash
   docker-compose exec mon1 ceph dashboard set-rgw-api-access-key -i /run/secrets/rgw_access_key
   docker-compose exec mon1 ceph dashboard  set-rgw-api-secret-key -i /run/secrets/rgw_secret_key
   ```

1. list buckets

   ```bash
   docker-compose exec mon1 radosgw-admin bucket list
   ```

## AWS CLI

```sh
docker-compose exec mon1 radosgw-admin user create --uid="deskoh" --display-name="Desmond Koh" --email="deskoh@example.org"
# Show user info (including access key and secret)
docker-compose exec mon1 radosgw-admin user info --uid=deskoh

aws configure --profile=ceph
aws --profile=ceph --endpoint=http://localhost:7480 s3api put-bucket-cors --bucket test --cors-configuration file://cors.json
aws --profile=ceph --endpoint=http://localhost:7480 s3 ls
aws --profile=ceph --endpoint=http://localhost:7480 s3 mb s3://test
```

## Reference

https://github.com/VasiliyLiao/ceph-docker-compose

https://docs.ceph.com/en/nautilus/mgr/dashboard/

https://www.fatalerrors.org/a/using-docker-to-build-ceph-cluster-nautilus-version.html

https://github.com/ceph/ceph/blob/master/src/sample.ceph.conf

https://github.com/ceph/ceph-container/blob/master/src/daemon/demo.sh
