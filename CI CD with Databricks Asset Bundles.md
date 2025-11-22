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
      
      - uses: databricks bundle deploy # we run the job
        working-directory: ./dab # setting the folder containing the bundle
        env:
          DATABRICKS_TOKEN: ${{secrets.SP_TOKEN}} # a tdatabricks token stored in github secrets
          DATABRICKS_BUNDLE_ENV: dev # databricks environment to be used

  run_pipeline_update:
    name: "Deploy bundle to DEV"
    runs-on: ubuntu-latest # environment to be used when executing the workflow
    environment: dev # environment name shown in github ui

    needs:
      - deploy # sets the dependency to the previous job

    steps:
      - uses: actions/checkout@v5 # checksout the code from github

      - uses: databricks/setup-cli@main # installing db cli
      
      - uses: databricks bundle run job_1 --refresh-all # we run the job
        working-directory: ./dab # setting the folder containing the bundle
        env:
          DATABRICKS_TOKEN: ${{secrets.SP_TOKEN}} # a tdatabricks token stored in github secrets
          DATABRICKS_BUNDLE_ENV: dev # databricks environment to be used
```