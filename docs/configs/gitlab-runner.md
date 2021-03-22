# Gitlab Runner
GitLab Runner is an application that works with GitLab CI/CD to run jobs in a pipeline.

Tu Graz Repository has a Gitlab group [invenio](https://gitlab.tugraz.at/invenio). That has a group runner [Gitlab-Runner-03-Produktion-Invenio-Shell](https://gitlab.tugraz.at/groups/invenio/-/settings/ci_cd?page=2#runners-settings), which is provided by [ZID](https://www.tugraz.at/tu-graz/organisationsstruktur/serviceeinrichtungen-und-stabsstellen/zentraler-informatikdienst/).

We are using this group runner to run our Pipeline jobs, from gitlab to our VM server's.


## How the GitLab runners are registered?

1. Access the node - for example: ```ssh mojib@instance_prod.tugraz.at``` then enter your password from [sesam.tugraz](https://sesam.tugraz.at) to gain access to the server.
2. [Add GitLab’s official repository](https://docs.gitlab.com/runner/install/linux-repository.html).
    
    ```curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash```


3. install the gitlab-runner

    ```apt install gitlab-runner```

4. Register the gitlab-runner for your group runner.

    ```gitlab-runner register```

      | Input           | Value|
      | --------------- | ----------- |
      | gitlab-ci URL:  | https://gitlab.tugraz.at/|
      | gitlab-ci token: | Check [sesam.tugraz](https://sesam.tugraz.at)|
      | gitlab-ci description: | invenio01-prod |
      | gitlab-ci tags: | prod-one |
      | executor: | shell |


5. Make sure to give permission.

    ```usermod -aG docker gitlab-runner```.

Runner is registered successfully. Now you should be able to see your Gitlab-runner in the list [here](https://gitlab.tugraz.at/groups/invenio/-/settings/ci_cd).

---

## Possible Errors
These error might apear when using this registered runner for the first time.

!!! error "The runner throws an ERROR"
    ERROR: Job failed (system failure): prepare environment: exit status 1
    [SEE HERE](https://docs.gitlab.com/runner/shells/index.html#shell-profile-loading).

**Solution**
Remove/comment ```.bash_logout``` entries.

- Access your VM server

- Navigate to ```/home/gitlab-runner```

- Look for file called ```.bash_logout```

- Remove or comment the entries on this file


!!! error "Error saving credentials"
    ```Error saving credentials: error storing credentials - err: exit status 1, out: `Cannot autolaunch D-Bus without X11 $DISPLAY```

**Solution:** install package in VM

```apt install gnupg2 pass```.


To learn more Gitlab-runner commands, visit
the [GitLab Runner commands](https://docs.gitlab.com/runner/commands/).
