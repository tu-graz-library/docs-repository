# CephFS  

The Ceph File System, or CephFS, is a POSIX-compliant file system built on top of Ceph’s distributed object store, RADOS.

## CephFS for repository

3 Folders were created for each environment:

1. Production environment - Quota 10TB

    ```
    /invenio/invenioprod: client 'fsinvenioprod', Secret in Sesam 
    ```

2. Test  environment - Quota 1TB

    ```
    /invenio/inveniotest: client 'fsinveniotest', Secret in Sesam 
    ```

3. Dev environment - Quota 1TB

    ```
    /invenio/inveniodev: client 'fsinveniodev', Secret in Sesam
    ```

---

In this guideline we will take a look on how the ```CEPH FS``` is configired for the production environment.
Which is also the same for ```Dev``` and ```Test``` environments.

### Steps

#### install ceph-common

```bash
apt install ceph-common
```

#### Create a file
Create a file, and add the secret from [sesam.tugraz](https://sesam.tugraz.at)

```bash
nano /etc/ceph/ceph.client.fsinvenioprod
```

#### Create a mount path
Create a directory which will be mounted to the CEPH FS.
```bash
mkdir /storage
```

#### Test mount
This will temporarily mount the directory ````/storage``` to the ```/invenio/invenioprod```

```bash
mount -t ceph <ip:port>:/invenio/invenioprod /storage -o name=fsinvenioprod,secretfile=/etc/ceph/ceph.client.fsinvenioprod
```

###### Check mounted file systems:
```bash
df -h
```
```console
Filesystem                                                                        Size  Used Avail Use% Mounted on
<ip:port>:/invenio/invenioprod   10T     0   10T   0% /storage
```

#### Unmount
After this temporarily mounting works, we will unmount it so later we can configure it properly.

```bash
umount /storage/
```

#### Add Mount configuration
open the ```/etc/fstab``` file and edit it as below:

```bash
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
<ip:port>:/invenio/invenioprod /storage  ceph _netdev,name=fsinvenioprod,secretfile=/etc/ceph/ceph.client.fsinvenioprod 0 0
```

#### Mount
```bash
mount /storage
```


