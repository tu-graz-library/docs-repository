# VMs update & upgrade

!!! warning "Backup"

    Make sure you have a regular backup of VMs set up! 


In total the repository uses 7 virtual machines, in order to keep the packages up to date, we need to run the `update` & `upgrade` commands in regular bases.

For this we have created a new repository [upgrade-vms](https://gitlab.tugraz.at/invenio/upgrade-vms), this repository consist of all the scripts and pipeline configuration to run our `update` & `upgrade` scripts in specified time and date, for all the VMs.

### .gitlab-ci.yml
is a YAML file that you create on your project's root. This file automatically runs whenever specific intervals are met.

This file consits of 7 stage, one stage for each virtual machines:

```yml
  - invenio-dev01
  - invenio01-test
  - invenio02-test
  - invenio03-test
  - invenio01-prod
  - invenio02-prod
  - invenio03-prod
```

All these above stages have the same identical steps, only the environment variables are different,
we will cover one of these stage.

```before_script``` run followings:

* Install ssh-agent if not already installed, it is required by Docker.
* Run ssh-agent (inside the build environment)
* Add the SSH key stored in TEST_01_PRIVATE_KEY variable to the agent store
* Create the SSH directory and give it the right permissions
* scan the keys of your private server from variable ```SSH_SERVER_HOSTKEYS```
```yml
  before_script:
    - echo "################################"
    - echo "SSH-USER"
    - eval $(ssh-agent -s)
    - echo "$TEST_01_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_SERVER_HOSTKEYS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
```

```script```:  using ```ssh``` runs scripts in ```scripts.sh``` to our server.
```yml
  script:
    - echo "################################"
    - ssh $TEST_01_DOMAIN "eval '$(cat ./scripts-test.sh)'"
```

Only run for branch ```schedules```:
```yml
  only:
    - schedules
```
Execute jobs with Gitlab-runner tag name ```shell```:
```yml
  tags:
   - shell
```

Full .gitlab-ci.yml:

```yml
invenio01-test:
  stage: invenio01-test
  before_script:
    - echo "################################"
    - echo "SSH-USER"
    - eval $(ssh-agent -s)
    - echo "$TEST_01_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_SERVER_HOSTKEYS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
  script:
    - echo "################################"
    - ssh $TEST_01_DOMAIN "eval '$(cat ./scripts-test.sh)'"
  only:
    - schedules
  tags:
   - shell
```

### scripts-().sh file
**scripts-test.sh** & **scripts-prod.sh** is a helper files for deployment, and used by
`.gitlab-ci.yml`. It contains instruction for update/upgrade commands.
And a `curl` check if nginx server is responding.

```bash
echo "################################################"
echo "Run update & upgrade scripts."

export DEBIAN_FRONTEND=noninteractive
apt-get update && apt-get -q -y upgrade


echo "################################################"
echo "Check if nginx server is running."

nginxserver=********:8080

check=$(curl -s -w "%{http_code}\n" -L "$nginxserver" -o /dev/null)
if [[ $check == 200 || $check == 403 ]]
then
    # Server is online
    echo "Server is online"
    exit 0
else
    # Server is offline or not working correctly
    echo "Server is offline or not working correctly"
    exit 1
fi

```


## Configure root access to VMS.
The `update` & `upgrade` shell commands require root access to the virtual machines.
We are using ssh-keys to access the virtual machines with root permission.

Please see the [SSH-key configuration](https://tu-graz-library.github.io/docs-repository/configs/sshkey/) for further details.

## Pipeline schedules
Now that we have our `scripts` and `pipeline configuration` ready, lets configure [Gitlab pipeline schedules](https://docs.gitlab.com/ee/ci/pipelines/schedules.html).

Gitlab pipelines are normally run based on certain conditions being met. For example, when a branch is pushed to repository.

Pipeline schedules can be used to also run pipelines at specific intervals. For example:

* Every month on the 22nd for a certain branch.
* Once every day.


### Configuring pipeline schedules
To schedule a pipeline for project:

1. Navigate to the project’s **CI/CD > Schedules** page.
2. Click the **New schedule** button.
3. Fill in the **Schedule a new pipeline** form.
4. Click the **Save pipeline schedule** button.

![](images/pipeline_schedules.png?raw=true)


In the **Schedules** index page you can see a list of the pipelines that are scheduled to run. The next run is automatically calculated by the server GitLab is installed on.
![](images/scheduled.png?raw=true)
