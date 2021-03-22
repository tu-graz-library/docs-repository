# Deploy

Ready to deploy your changes in one of our instances:

In this guideline, we will take a look at when the pipeline for different instances is executed.

1. Deploy Dev ([Development instance](https://invenio-dev01.tugraz.at)) 
2. Deploy Test ([Test instance](https://invenio-test.tugraz.at))
3. Deploy Production ([Production instance](https://repository.tugraz.at))

## Deploy Dev
Every time there is a branch ```dev``` or ```merge_request``` to the branch master of **[Repository](https://gitlab.tugraz.at/invenio/repository)**
the pipeline stage for ```dev``` is executed.

### Pipeline
![](images/pipeline-dev.JPG?raw=true)

<br />

## Deploy Test
Every **commit** or **merge** to the master branch of **[Repository](https://gitlab.tugraz.at/invenio/repository)** will run Pipeline and
deploy the changes to the [Test instance](https://invenio-test.tugraz.at).

### Pipeline
![](images/test-pipeline.JPG?raw=true)

<br />

## Deploy Production
Every new ```Tag/release``` of the **[Repository](https://gitlab.tugraz.at/invenio/repository)** will run Pipeline and 
deploy the changes to the [Production instance](https://repository.tugraz.at).

### Steps
Deployment to production requires a new ```Tag/release``` of the **[Repository](https://gitlab.tugraz.at/invenio/repository)**.
Meaning we should only deploy to production when we have a new ```Tag/release```.

**1.** Before creating a new ```Tag/release``` first we must change the value of ```TAG_PROD``` in [GitLab CI/CD variables](https://docs.gitlab.com/ee/ci/variables/), to our expected new ```Tag/release``` version.
   
   For example the following change in ```TAG_PROD``` variable:

```diff
- TAG_PROD=v0.1.1 
+ TAG_PROD=v0.1.2
```

**2.** Create a new ```Tag/release``` for the **[Repository](https://gitlab.tugraz.at/invenio/repository)**, using semantic-versioning.
   for example ```v0.1.2```, same value as in the **.env** file.

   For Example using git:

```
git tag -a v0.1.2 -m "my version 0.1.2"
 ```

### Pipeline
![](images/prod-pipeline.JPG?raw=true)
