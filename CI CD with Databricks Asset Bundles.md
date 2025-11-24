# Prerequirements

- Create an accoun at Databricks Free Edition
- Install Databricks CLI with
    ```linux
    sudo curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sudo sh
    ```
- On your Databricks workspace, create a token
- Use that token to authenticate running
    ```linux
    databricks configure
    ```

# Asset Bundles
Asset Bundles are used in Databricks to create CI/CD workflows that allow you to test your code, create artifacts and deploy them to different environments

## Create a Databricks Asset Bundle
In your selected folder, go on and create folder and run on it:
```linux
databricks bundle <your_folder>
```

After setting the configuration, this will create a project folder, including a `databricks.yml` that is the configuration of your bundle.

## Deploy your Asset Bundle
Move to the folder of your project where the `databricks.yml` file exists and run:
```linux
databricks bundle deploy
```

This will deploy the bundle to te Databricks Workspace. It will deployed as a job in the 'Workflows' tab.

## DAB main CLI commands
Here are the main commands for DAB in Databricks CLI:

- `databricks bundle validate`: executes the validation checks on your `databricks.yml` before deploying it
- `databricks bundle summary`: will show a summary of the current bundle from the `databricks.yml`
- `databricks bundle deploy`: deploy the bundle
- `databricks bundle deploy -t <environment>`: deploy the bundle to a certain environment
- `databricks bundle open <job name>`: this will open the bundle on the Databricks UI
- `databricks bundle destroy <job name>`: delete a bundle
- `databricks bundle destroy <job name>-t <environment>`: delete a bundle in a certain environment
- `databricks bundle run <job name>`: runs a bundle. Starts a cluster and terminates it after running the job

## How to find information about YML Structure of Asset Bundle
You can access the JSON of the bundle from clicking on the job name on the `databricks.yml` file. There you can check the configuration of everything on the `.yml` file.

You may also create the job manually from the UI and then copy the `.yml` to your bundle.

## Running Multiple Tasks in Asset Bundles
Witihn your project folder there is a folder 'resources' that contains the configuration of the job:

- `databricks.yml` contains the configration of the project, such as the Databricks' Workspaces and environments
- `resources/<your_project>.job.yml`: contains the yml configuration of the job itself, with the tasks and everything.

On `resources/<your_project>.job.yml` you can add new tasks. If you don't use the command `depends_on` they will be executed concurrently.

## Running Multiple Tasks Sequentially in Asset Bundle

On `resources/<your_project>.job.yml` you can add new tasks. If you don't use the command `depends_on` they will be executed concurrently. Otherwise, you can set dependencies.

You may also use the key word 'run_if' to set when a tak should run based on its dependencies:
```yml
tasks:
        - task_key: notebook_task_1
          notebook_task:
            notebook_path: ../src/sample_notebook.ipynb
        - task_key: notebook_task_2
          notebook_task:
            notebook_path: ../src/sample_notebook_copy.ipynb
        - task_key: notebook_task_3
          depends_on:
            - task_key: notebook_task_1
            - task_key: notebook_task_2
          run_if: ALL_DONE
          notebook_task:
            notebook_path: ../src/sample_notebook_copy.ipynb
```

## DAB: Setting Variables
Normally we will have different configurations for deifferent environments. We can set them in the `databricks.yml`

This file uses bash stiling or variables `$()`. You can refer them using this.

When you run a command that takes input parameters, they will be fetched and used in the varables.

Other than this, we can set variables with:

```yml
variables:
  catalog:
    description: The catalog to use
  schema:
    description: The schema to use
```

This variables may be referred from the `databricks.yml` file but also from your project yml. You can refer them like `$(var.<your variable>)`

For instance, you may define a notebook pat in `databricks.yml`:
```yml
variables:
  dab_notebook: 
    default: ../src/sample_notebook.ipynb
```

And use it to reference your noteooks from your project yml:
```yml
tasks:
        - task_key: notebook_task_1
          notebook_task:
            notebook_path: ${var.dab_notebook}
```

Note that the path used a relative path, because it will use the path of your project yml as baseline.

Moreover, you can set a variable for each specific environment. Inside the environment section, use a new `variables` section and define a variable with the same name to override the default value:

```yml
bundle:
  name: init_project
  uuid: 2bd27268-8d69-4565-bf5e-7fe2fb3e8bf6

variables:
  dab_notebook: 
    default: ../src/sample_notebook.ipynb

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://dbc-bd92c4a5-086d.cloud.databricks.com
  prod:
    mode: production
    workspace:
      host: https://dbc-bd92c4a5-086d.cloud.databricks.com
      # We explicitly deploy to /Workspace/Users/guzman.garcia1990@gmail.com to make sure we only have a single copy.
      root_path: /Workspace/Users/guzman.garcia1990@gmail.com/.bundle/${bundle.name}/${bundle.target}
    variables:
      dab_notebook: ../src/sample_notebook.ipynb

```

## Override Variables
When you create a varable, you define a default option that can be overriden.

Other than setting an environment variable in your target, you may override this variabls with input parameters when you run your bundle with:
```linux
databricks bundle run -o json --var="<your_var_defined_in_databricks.yml>="your_value"
```

## Complex variables
The course use a cluster as an example of a complex variable (since I am using the Databricks Free Version, clusters are not available).

However, the point is that in your project yml you can add a key word `job_cluster` to choose which cluster to use.

Now this is complex because:
- You may want to use different clusters for different tasks
- You definetely need to use different clusters for different environments.

So, if you have five different tasks and want to use five different clusters, we can use complex variables.

Go ahead and within your main project folder create a folder named `clusters` and inside crete a yml file, for instance `dab_cluster.yml`.

You can use this for the contents:

```yml
# dab_cluster.yml
varibales:
    dab_cluster_task_1:
        description: "My DAB Cluster task 1"
        type: complex
        default:
          spark_version: 15.4.x-scala2.12
          node_type: Standard_D3_v2
          autoscale:
            min_workers: 2
            max_workers: 4
        spark_conf:
            spark_speculation: true
    
    dab_cluster_task_2:
        description: "My DAB Cluster task 2"
        type: complex
        default:
          spark_version: 15.4.x-scala2.12
          node_type: Standard_D3_v2
          autoscale:
            min_workers: 2
            max_workers: 7
        spark_conf:
            spark_speculation: true
```

Then, back to your main project yml file, you can go and use these clusters:
```yml
# your_project.job.yml
    tasks:
        - task_key: notebook_task_1
          job_cluster_key: job_cluster_1 # using the cluster variable below defined
          notebook_task:
            notebook_path: ${var.dab_notebook}
        - task_key: notebook_task_2
          job_cluster_key: job_cluster_2 # using the cluster variable below defined
          notebook_task:
            notebook_path: ../src/sample_notebook_copy.ipynb
    # we define variables in this file that refer to the variables in the clusters/dab_cluster.yml
    job_clusters:
        - job_cluster_1: job_cluster
        new_cluster: ${var.dab_cluster_task_1}
        - job_cluster_2: job_cluster_2
        new_cluster: ${var.dab_cluster_task_2}
```

Now, there is something still missing, and that is for us to be able to refer to the contents of our `clusters/dab_cluster.yml` from `your_project.job.yml`. For that, we need to edit the `databricks.yml` file:

```yml
# databricks.yml
# on the top of the file include a section like this
include:
  - resources/*.yml
  - clusters/*.yml
```

Now we can use the files defined in `clusters/` in our `your_project.job.yml` file.

We could also create configuration for each environment creating a folder `environments/`.

## Retrieval Variables
Wth thi functionality can get the values from objects. For instance, retrieve a cluster id and use it in our tasks.

In our `clusters/dab_cluster.yml` let's add a new variable.

```yml
# dab_cluster.yml
varibales:
    dab_cluster_task_1:
        description: "My DAB Cluster task 1"
        type: complex
        default:
          spark_version: 15.4.x-scala2.12
          node_type: Standard_D3_v2
          autoscale:
            min_workers: 2
            max_workers: 4
        spark_conf:
            spark_speculation: true
    
    dab_cluster_task_2:
        description: "My DAB Cluster task 2"
        type: complex
        default:
          spark_version: 15.4.x-scala2.12
          node_type: Standard_D3_v2
          autoscale:
            min_workers: 2
            max_workers: 7
        spark_conf:
            spark_speculation: true
    
    # adding a new cluster configuration, so that we use an existing cluster in our jobs
    existing_cluster:
        description: "My existing Cluster task 2"
        lookup:
            cluster: "your_existing_cluster_name"
```

And back to `your_project.job.yml`:

```yml
# your_project.job.yml
    tasks:
        - task_key: notebook_task_1
          job_cluster_key: job_cluster_1 # using the cluster variable below defined
          notebook_task:
            notebook_path: ${var.dab_notebook}
        - task_key: notebook_task_2
          existing_cluster_id: ${var.existing_cluster} # using an existing cluster
          notebook_task:
            notebook_path: ../src/sample_notebook_copy.ipynb
    # we define variables in this file that refer to the variables in the clusters/dab_cluster.yml
    job_clusters:
        - job_cluster_1: job_cluster
        new_cluster: ${var.dab_cluster_task_1}
```

*To use a job cluster we need to use the key word* **job_cluster_key**.

*To use an existing cluster we need to use the key word* **existing_cluster_id**.

## Pasing Simple Parameters
**When running your bundle you can pass parameters**:
- ***on the job level***
- ***on the task level***

### Parameters on the Task Level

You can edit `your_project.job.yml` to look like:
```yml
    tasks:
        - task_key: notebook_task_1
          notebook_task:
            notebook_path: ${var.dab_notebook}
            base_parameters:
              notebook_param: "my_param"
```

So, basically, we add a key `base_param` and within we create our paramters. These can be used in the notebook.

#### Passing Parameters between Tasks
Altough this is initially not possible, the course shows a bypass for this.

We can use this in our notebook outputting the paramter:
```python
dbutils.jobs.taskValues.set(key='from_t2', value=new_notebook_param)
```

And this in the task receiving the parameter:
```yml
# your_project.job.yml

tasks:
        - task_key: notebook_task_1
          notebook_task:
            notebook_path: ${var.dab_notebook}
        - task_key: notebook_task_2
          notebook_task:
            notebook_path: ../src/sample_notebook_2.ipynb
            base_parameters:
              notebook_param: "my_param"
        - task_key: notebook_task_3
          depends_on:
            - task_key: notebook_task_1
            - task_key: notebook_task_2
          run_if: ALL_DONE
          notebook_task:
            notebook_path: ../src/sample_notebook_3.ipynb

            ## adding this on the base paramters, that refers to the output of task notebook_task_2 and the key outputed in dbutils.jobs.taskValue.setKeys
            base_parameters:
              from_t2: "{{tasks.notebook_task_2.values.from_t2}}"
```

Now, finally, we can use this passed value as input in the notebook receiving it with a `dbutils.widget.get`.

### Adding Spark Variables
We can pass tailored Spark config variables to our jobs. Goin back to our `clusters/dab_cluster.yml`:
```yml
# clusters/dab_cluster.yml
varibales:
    dab_cluster_task_1:
        description: "My DAB Cluster task 1"
        type: complex
        default:
          spark_version: 15.4.x-scala2.12
          node_type: Standard_D3_v2
          autoscale:
            min_workers: 2
            max_workers: 4
        spark_conf:
          spark_speculation: true
          partitionDate: ${var.partition_date}
        spark_env_vars:
          partitionDate: ${var.partition_date}
```

And we can go ahead and create the variable in our `databricks.yml`:
```yml
# databricks.yml

variables:
  catalog:
    description: The catalog to use
  schema:
    description: The schema to use
  dab_notebook: 
    default: ../src/sample_notebook_1.ipynb
  partition_date: # adding the variable used in clusters/dab_cluster.yml
    default: "2024-01-01"
```

Provided wew want to use the variable in a notebook:
```python
partition_date = spark.conf.get('partitionDate')
print(partition_date)

# or
import os
partition_date = os.getenv('partitionDate')
```

### Parameters on the Job Level
Let's go and create an extra job in `resources/`:
 - `job_1.job.yml`
 - `job_2.job.yml`

Now, we want to add a task in `job_1.job.yml` to triger the second job:

```yml
# job_1.job.yml
tasks:
    - task_key: job_2_trigger
      depends_on: 
        - task_key: your_dependency
      run_job_task:
        job_id: ${resources.jobs.job_2.id}
        job_parameters:
            job_param: "{{tasks.job_1_notebook_task_3.values.job_param}}"
```

Now, in the notebook that we cant to input the paramter into the second job we use `dbutils.jobs.taskValues.set`, and in the notebook receiving this we use our usual `dbutils.widgets.get`

In `job_2.job.yml` we simply create the task:
```yml
# bob_2.job_yml
# A sample job for init_project.

resources:
  jobs:
    job_2:
      name: job_2

      parameters:
        - name: catalog
          default: ${var.catalog}
        - name: schema
          default: ${var.schema}

      tasks: 
        - task_key: job_2_task_1
          run_if: ALL_DONE
          notebook_task:
            notebook_path: ../src/sample_notebook_4.ipynb
          ## no parameters need to be defined here, it will simply fetch it from the previous notebook and work with dbutils.widgets.get in the execution notebook

      environments:
        - environment_key: default
          spec:
            environment_version: "4"


```

## Implementation CI/CD GitHub Action pipeline

### Implementing simple "on push" pipeline
This lesson shows how to automate the bundle runs through a Github pipeline.

Within our project folder, we are creating a new folder `.github/` and within yet another `workflows`.

Finally, within `.github/workflows/` we create `cicd.yml`

This workflow will deplo the jobs, and then run them. It will use databricks token that needs to be stored in Github -> Repo -> Settings -> Environment.

This is how it should look:
```yml
# name of your workflow
name: "DEV deployment"

# number of allowed concurrent executions
concurrency: "1"

# event trigering the workflow
# this will trigger the pipeline on pishing changes
on: [push]

# jobs to be executed on action
jobs:
  deploy:
    name: "Deploy bundle to DEV"
    runs-on: ubuntu-latest # environment to be used when executing the workflow
    environment: dev # environment name shown in github ui

    steps:
      - uses: actions/checkout@v5 # checksout the code from github

      - uses: databricks/setup-cli@main # installing db cli
      
      - run: databricks bundle deploy # we run the job
        working-directory: ./init_project/dab # setting the folder containing the bundle
        env:
          DATABRICKS_TOKEN: ${{secrets.SP_TOKEN}} # a tdatabricks token stored in github secrets
          DATABRICKS_BUNDLE_ENV: dev # databricks environment to be used

  run_pipeline_update:
    name: "Run pipeline in DEV"
    runs-on: ubuntu-latest # environment to be used when executing the workflow
    environment: dev # environment name shown in github ui

    needs:
      - deploy # sets the dependency to the previous job

    steps:
      - uses: actions/checkout@v5 # checksout the code from github

      - uses: databricks/setup-cli@main # installing db cli
      
      - run: databricks bundle run job_1 --refresh-all # we run the job
        working-directory: ./init_project/dab # setting the folder containing the bundle
        env:
          DATABRICKS_TOKEN: ${{secrets.SP_TOKEN}} # a tdatabricks token stored in github secrets
          DATABRICKS_BUNDLE_ENV: dev # databricks environment to be used
```

### Implementing a "pull request" pipeline
Thi will basically be the same, but we need to change the `cicd.yml` file to use the correct configuration on the trigger action:
```yml
# cicd.yml
# name of your workflow
name: "DEV deployment"

# number of allowed concurrent executions
concurrency: "1"

# event trigering the workflow
# this will trigger the pipeline on pull request changes
on:
  pull_request:
    branches: [master]

# jobs to be executed on action
jobs:
  deploy:
    name: "Deploy bundle to DEV"
    runs-on: ubuntu-latest # environment to be used when executing the workflow
    environment: dev # environment name shown in github ui

    steps:
      - uses: actions/checkout@v5 # checksout the code from github

      - uses: databricks/setup-cli@main # installing db cli
      
      - run: databricks bundle deploy # we run the job
        working-directory: ./init_project/dab # setting the folder containing the bundle
        env:
          DATABRICKS_TOKEN: ${{secrets.SP_TOKEN}} # a tdatabricks token stored in github secrets
          DATABRICKS_BUNDLE_ENV: dev # databricks environment to be used

  run_pipeline_update:
    name: "Run pipeline in DEV"
    runs-on: ubuntu-latest # environment to be used when executing the workflow
    environment: dev # environment name shown in github ui

    needs:
      - deploy # sets the dependency to the previous job

    steps:
      - uses: actions/checkout@v5 # checksout the code from github

      - uses: databricks/setup-cli@main # installing db cli
      
      - run: databricks bundle run job_1 --refresh-all # we run the job
        working-directory: ./init_project/dab # setting the folder containing the bundle
        env:
          DATABRICKS_TOKEN: ${{secrets.SP_TOKEN}} # a tdatabricks token stored in github secrets
          DATABRICKS_BUNDLE_ENV: dev # databricks environment to be used

```

After changing this, go back to Github -> Repo -> Settings -> Rulesets  and create a new rule. This is done to add extra security on your PR. Basically we will be setting master as target branch, and activating te options "Require a pull request before merging" and "Require status checks to pass" (add your stages in your yml pipeline in this step so that they are triggered on the PR before merging). This will check the pipeline works BEFORE merging.

After this, you are good to go to commit, PR and merge. If set correctly, after the PR and before the merge, the stages added on "Require status checks to pass" will be triggered. If executed correctly, you will be able to merge.

## Implementing CI/CD Azure

### Azure Devops Pipeline with regular Databricks Token
Create a new repo on Azure Devops, clone it to your local and initialize a databricks bundle with `databricks bundle init`.

Then, create an `azure_pipeline.yml` within your project `your_project/` folder with:

```yml
trigger: none

pool:
  vmImage: ubuntu-latest

variables:
  - group: 'DatabricksSettings'
  - nam: SOURCE_DIR
    value: 'DAB/devops_bundle'

steps:
# Install Dataricks CLI
- script: |
    if ! command -v databricks &> /dev/null; then
      curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh |
      echo "Databricks CLI instaled"
  displayName: Databricks CLI Installation

# Run DAB validation
- script: |
    databricks bundle validate
  workingDirectory: $(SOURCE_DIR)
  displayName: DAB validation
  # env
  #   DATABRICKS_HOST: $(DATABRICKS_HOST)
  #   DATABRICKS_TOKEN: $(DATABRICKS_TOKEN)

- script: |
    databricks bundle deploy
  workingDirectory: $(SOURCE_DIR)
  displayName: DAB Deployment
  # env
  #   DATABRICKS_HOST: $(DATABRICKS_HOST)
  #   DATABRICKS_TOKEN: $(DATABRICKS
```

Back to your Azure Devops Workspace go to Pipelines -> Library and create a new variable group with name `DatabricksSettings` (matching the name in your azure_pipeline.yml file for the group) and create two variables `DATABRICKS_HOST` and `DATABRICKS_TOKEN` (with your account details).

After this, you can create a new pipeline and select your `azure_pipeline.yml` file.

### Azure DevOps pipeline with Azure Service Principal
The best practice is to use a **Service Principal** instead of a user: users go on hollidays and leave te company. A Service principal is a placeholder solution that allows to connect to your resources regardless of the user executing the action.

To do this you need to go to Azure Portal -> Entra ID -> App registration.

Create a name for your Server Principal and register it.

Once registered, collect the following details:
- Application ID
- Azure Tenant
- Create a new client secret (name it 'Databricks Access')

Back to Databricks go to accounts: Databriks accounts.azuredatabricks.com
Accounts -> User Management -> Service Principals -> Add your Service Principal (use your client id from Azure). Now you have an account not connected to any workspace that you can use or your Devops endeavors.

Back to the workspace -> Settings -> should display an additional tab for Workspace admin -> Identity and access -> Service Principals -> Add your service principal (should be displayed as an option in the dropdown).

Now you can use the Service Principal in your pipeline.

So the overall process is:
- Setting up your Azure Account
- Setting your Tenant
- Setting your Devops Workspace and connect it to your Tenant with Microsoft Entra ID
- In Microsoft Entra ID, create a new Service Principal and fetch its ID and key apart from the Azure Tenant
- Go to accounts.dataricks.net or accounts.databricks.com and set your service principal connection
- Go to your Databricks Workspace -> Settings -> Workspace Settings -> Identity and access -> Add your service principal to the Workspace

After this, we may go back to DevOps and add the Service Principal to our Variable group. Instead of the databricks Host and Token, we will add:
- ARM_TENANT_ID: the Azure Tenant
- ARM_CLIENT_ID: the Service Principal ID
- ARM_CLIENT_SECRET: the Service Principal secret
- DATABRICKS_AZURE_RESOURCE_ID: here we need to pass the id workspace fetched from the Azure Portal -> resource -> JSON view -> Copy the Resource ID

With all these details, the pipeline will now look like this:
```yml
# Azure Devops pipeline with Service Principal
trigger: none

pool:
  vmImage: ubuntu-latest

variables:
  - group: DatabricksSettings
  - name: SOURCE_DIR
    value: 'DAB/devops_bundle'

steps:

# Install Databricks CLI
- script: |
    curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh | sh
  displayName: Databricks CLI Installation

# Run DAB validation
- script: |
    databricks bundle validate
  workingDirectory: $(SOURCE_DIR)
  displayName: DAB validation
  env:
    DATABRICKS_AZURE_CLIENT_ID: $(ARM_CLIENT_ID)
    DATABRICKS_AZURE_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
    DATABRICKS_AZURE_TENANT_ID: $(ARM_TENANT_ID)
    DATABRICKS_AZURE_RESOURCE_ID: $(DATABRICKS_AZURE_RESOURCE_ID)

# Run DAB deployment
- script: |
    databricks bundle deploy
  workingDirectory: $(SOURCE_DIR)
  displayName: DAB Deployment
  env:
    DATABRICKS_AZURE_CLIENT_ID: $(ARM_CLIENT_ID)
    DATABRICKS_AZURE_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
    DATABRICKS_AZURE_TENANT_ID: $(ARM_TENANT_ID)
    DATABRICKS_AZURE_RESOURCE_ID: $(DATABRICKS_AZURE_RESOURCE_ID)
```

## Python notebook end-to-end project
### Setup Unity Catalog for Azure Databricks Service
- First, you need to create a storage account in Azure to set it as your storage for your Unity Catalog. Look for storage account in Azure Portal and set it (set the Cold Accesss tier).

- Now, we need to connect the Databricks Virtual Network to the created Storage Account. Go to the Storage Account and create a container (name it what you like, for instance 'metastore').

- To connect the container, look for Access Connector for Azure Databricks on the Portal, select the resource group where your Databricks resource is, and make sure that the access connector uses the same region as the Datbrciks resource. Create it.

- Grant the necessary permission to your Access Connector. Azure Portal -> Storage Account -> Access Control -> Add Role Assignment -> Grant Storage Blob Contributor -> Assign Access to 'Managed Identity' -> Select your Access Conector for Azure Databricks

- After creating the connector, go to Databricks -> Manage Account (accounts.databricks.net or accounts.databricks.com) -> Catalog -> Create a metastore (on the same region) -> connect it to the created container with the indications provided in the page -> Input the Access Connector ID

- After creating the metastore in Databricks, assign it to the workspace (it is inmediately prompted after the creation). CLick on 'Enable Unity Catalog'

- Unity Catalog is now set. You can access 'Catalog' on the side pane and create schemas and volumes.

## Python Wheel Projects
A Wheel is a package you can use to create libraries to deploy them to a cluster.

This is veryy convenient to install custom libraries in several Databricks Workspaces.

### DAB wheel simple
Let's showcase a simple package deployment. This is merely a showcase and is not production ready

A Wheel is a package you can use to create libraries to deploy them to a cluster.

Create a new databricks asset bundle repo, for instance 'wheel_package', and deploy it. Mke sure you choose the package option when creating it. To deploy the bundle you'l need to have installed uv, you may install it with `sudo snap install astral-uv --classic`

In Databricks -> Workspace -> User -> .bundle/wheel_packae/dev/artifacts/ you'll finde a file containing th package.

Back to the workflows, you can see the flow and run it.

In your local, at `resources/<your_job_name>.job.yml`, you'll have to configure your job and package. Check the tasks section:

```yml
tasks:
        - task_key: notebook_task
          notebook_task:
            notebook_path: ../src/sample_notebook.ipynb
        - task_key: python_wheel_task
          depends_on:
            - task_key: notebook_task
          python_wheel_task:
            package_name: wheel_package
            entry_point: main
            parameters:
              - "--catalog"
              - "${var.catalog}"
              - "--schema"
              - "${var.schema}"
          environment_key: default
          libraries:
            ## Add your libraries to this folder
            - whl: ../dist/*.whl
```

In the libraries part in the main wheel task you need to install everything in the folder figuring there. Your wheel package file is also there. So you need to install every requirement in that folder.

### DAB wheel with Poetry
We can use poetry as virtual environment manager and package manager. Install it with `sudo apt install python3-poetry`.

Then, create a new bundle with `databricks bundle init`.

Now, let's modify the configuration to enable poetry. If a `setup.py` file wa created, remove it. 

Let's create a virtual environment and add poetry. Move to your bundle folder and run `pyton3 -m ven venv` to create your virtual environment and activate it with `source venv/bin/activate`.

Now, let's activate poetry with `poetry init`.

With this in place,run `poetry env info` to check the environment used by poetry.

If everything seems fine, we can modify the `databricks.yml` file like:
```yml
artifacts:
  python_artifact:
    type: whl
    build: poetry build
    path: .
```

Now, in your `resources/<your_project>.job.yml` you'll see a task refering to the wheel:
```yml
- task_key: python_wheel_task
          depends_on:
            - task_key: notebook_task
          python_wheel_task:
            package_name: wheel_poetry
            entry_point: main
```
Basically, add the correct python artifact.

The defined entry point there needs to be refenrenced by poetry somehow. This is done in **`pyproject.toml`**, where you need to add:
```
[tool.poetry.scripts]
main = "wheel_poetry.main:main"
```
***This will allow the yml file to use main as entry point so that the package can be build using poetry*** It works the following way: on wheel_project go to the main file and execute the main function.


Now, you can go on and do your development and add your dependencies with `poetry add <library_name>`. The command `poetry build .` will be executed when running `databricks bundle run` and will create the wheel package.

### DAB wheel multi package with Poetry.
Custom libraries can also be developed following this approach. Furthermore, you can have several wheel pacakges in a single bundle.

To do that, yo can create as many modules as needed in folders. Then, you need to run the `poetry init` in each one of them so that the `pyproject.toml` file is generated for each module. Then, run `poetry build` to enerate the `.whl` files in each module. For this to wor, first you'll have to update the `pyproject.toml` with the dependencies if necesary. This means, if one of your modules depends on the others, you need to manually add the depdendencies. For instance, for a module having a dependency on another module called `mymath`, you'll have to add:
```
# pyproject.toml
dependencies = [
  "mymath>=0.1.0", # assuming this is the version
]
```

After this, make sure to update the `databricks.yml` file like this:
```yml
artifacts:
  <your_artifact_name>: # this is your module
    type: whl
    build: poetry build
    path: <path from where the databricks.yml file is found>
```

And your job `<your_project>.job.yml`:
```yml
# resources/<your_project>.job.yml

tasks:
  - task_key: notebook_task # placeholder name
    job_cluster: <your_cluster>
    notebook_task:
      notebook_path: <relative path from the path of the current file to the path where notebook is found>
    libraries:
      - whl: <path to your library/module whl file>
```

### DAB wheel package with UV
Similarly, we can use UV for our packages in asset bundle. Create a new bundle and add this to the `databricks.yml` file:
```yml
artifacts:
  python_artifact:
    type: whl
    build: uv build --wheel
```

Or, if you have several artifacts, do it like for the poetry example:
```yml
artifacts:
  <your_artifact_name>: # this is your module
    type: whl
    build: uv build --wheel
    path: <path from where the databricks.yml file is found>
```
Note that for several modules and pyprojects, the same steps as in the previous section will have to be followed.

The rest remains the same as for poetry.