#### Given:
- 3 machines with a raised Ceph cluster. Disk damaged on ceph3
- A prefabricated ceph4 OSD node
- Client machine

#### 1. Restore the cluster
#### 1.1 Add ceph4 host to cluster in `/etc/hosts` in ceph1 and ceph4

`<ip_ceph4> ceph4.local ceph4`

#### 1.2 Add ssh key to ceph4

`ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph4`

#### 1.3 Add disk ceph4 to cluster

`ceph orch host add ceph4 <ip_ceph4>`

`ceph orch daemon add osd ceph4:/dev/<disk>`

#### 1.4 Find the failed disk and remove it from crushmap and c osd and corresponding daemon

#### Determine his name

`ceph orch ps`

#### Delete disk

`ceph osd rm <name_disk>`

#### Remove entry from crushmap

`ceph osd crush remove <name_disk>`

#### Remove the daemon responsible for the failed drive

`ceph orch daemon rm <name_disk> --force`

#### 2. Raise RGW `realm=ceph.local` `rgw-zone=ceph.local` everything else defaults (radosgw-admin). Gateway machines: ceph1 and ceph2

#### 2.1 Create realm

`radosgw-admin realm create --rgw-realm=ceph.local --default`

#### 2.2 Create zone

`radosgw-admin zonegroup create --rgw-zonegroup=default --master --default`

#### Create group

`radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=ceph.local --master --default`

#### 2.3 Commit changes and orchestrate containers on ceph1 and ceph2

`radosgw-admin period update --rgw-realm=ceph.local --commit`

#### Orchestrating containers

`ceph orch apply rgw ceph --realm=ceph.local --zone=ceph.local --placement="2 ceph1 ceph2"`

#### 2.4 Create user test (ragosgw-admin)

`radosgw-admin user create --display-name="test" --uid=test`

##### *Take this data for aws creds*

```
	    "user": "test",
            "access_key": "<ACCESS_KEY>,
            "secret_key": "<SECRET_KEY"`
```            

#### 3. Setting client machine

#### 3.1 Install awscli, boto, boto3

```
apt update
apt install python3-pip -y
pip3 install awscli boto boto3
```
### 3.2 Configure the client using the keys received from the cluster - the ceph profile

`aws configure --profile=ceph`

#### 3.3 Check the settings and access to the gateway and create an s3-bucket  with name `backup`

`cat ~/.aws/credentials`

`aws --profile=ceph --endpoint=http://ceph1 s3 ls`

`aws --profile=ceph --endpoint=http://ceph1 s3api create-bucket --bucket backup`

`aws --profile=ceph --endpoint=http://ceph1 s3 ls`

#### Create file in s3-bucket backup

`touch test`

`aws --profile=ceph --endpoint=http://ceph1 s3 cp test s3://backup/`

#### Check

`aws --profile=ceph --endpoint=http://ceph1 s3 ls s3://backup/
