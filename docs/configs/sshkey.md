# SSH-key

[Services VM](http://localhost:8000/deployment/#services-vm)
deployment is using direct ```ssh``` commands to reach the VM. In order to know the ssh keys configuration, please take a look into the steps below:

### Generate ssh-key in VM
This command will generate a new ssh-key.

```bash
$ ssh-keygen
```

```console
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
```

### Add private key to CI/CD variable

we have defined a variable ```PROD_03_PRIVATE_KEY``` in our Gitlab [invenio group variable](https://gitlab.tugraz.at/groups/invenio/-/settings/ci_cd).
And we are adding the generated private key ```/root/.ssh/id_rsa``` to ```PROD_03_PRIVATE_KEY```.

### authorized_keys 
The authorized_keys file in SSH specifies the SSH keys that can be used for logging into the user account for which the file is configured.

Add public key to authorized_keys:

```bash
$ cat  /root/.ssh/id_rsa.pub >> /.ssh/authorized_keys
```

### known_hosts
group runner [Gitlab-Runner-03-Produktion-Invenio-Shell](https://gitlab.tugraz.at/groups/invenio/-/settings/ci_cd?page=2#runners-settings) has a ```known_hosts``` that contains a list of public keys for all the hosts which the user has connected to.

These list of public keys are in CI/CD variable ```SSH_SERVER_HOSTKEYS```.

To add new host:
```bash
$ ssh-keyscan invenio03-prod.tugraz.at
```
And then update the variable ```SSH_SERVER_HOSTKEYS```.

