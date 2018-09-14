Author: Joseph Barron

# Circle CI
CircleCI uses a YAML file to identify how you want your testing environment set up and what tests you want to run.

## Testing Config Files Locally
See [this page](https://circleci.com/docs/2.0/examples/).

## How do Docker image names work?
CircleCI 2.0 currently supports pulling/pushing Docker images from Docker Hub. For [official images](https://hub.docker.com/explore/), you can pull by simply specifying the name of the image and a tag:
```
golang:1.7.1-jessie
redis:3.0.7-jessie
```
For public images on Docker Hub, you can pull the image by prefixing the account or team username:
```
myUsername/couchdb:1.6.1
```

# Diagram of Circle CI Workflows
![This is heckin' useful.](https://circleci.com/docs/assets/img/docs/Diagram-v3-Workspaces.png)

# Configuration
See the [configuration reference](https://circleci.com/docs/2.0/configuration-reference/).

## `version`
Should be 2.

## `jobs`
A run is comprised of one or more named jobs. Jobs are specified in the `jobs` map. The name of the job is the key in the ma, and the value is a map describing the job.

If you are not using *Workflows*, the `jobs` map must contain a job named `build`, which is the default entry-point for a run triggered by a push.

**Note:** Jobs have a maximum runtime of 5 hours.

**Example:**
```YAML
version: 2
jobs:
  build:
    docker:
      - image: circleci/<language>:<version TAG>
    steps:
      - checkout
      - run: <command>
  test:
    docker:
      - image: circleci/<language>:<version TAG>
    steps:
      - checkout
      - run: <command>
workflows:
  version: 2
  build_and_test:
    jobs:
      - build
      - test
```

### &lt;`job_name`&gt;
Each job consists of the job's name as a key and a map as a value. A name should be unique within a current `jobs` list.

Requires options for at least one of: `docker`, `machine`, or `macos`.

Requires `steps`, a list of *steps* to be performed.

Optional `environment`, a map of environment variable names and values.

Optional `branches`, a map defining rules for whitelisting/blacklisting execution of specific branches for a single job that is not in a workflow.

#### `docker` / `machine` / `macos` (executor)
An "executor" is roughly a "place where steps occur". 

#### `docker`
Configured by `docker` key which takes a list of maps.

Requires `image`, a string name of a custom docker image to use. If you are using a private image, you can specify the username and password in the `auth` field. To protect the password, you can set it as a project setting which you reference here:
```YAML
jobs:
  build:
    docker:
      - image: acme-private/private-image:321
        auth:
          username: mydockerhub-user  # can specify string literal values
          password: $DOCKERHUB_PASSWORD  # or project UI env-var reference
```

Optional `user`, which user to run commands as wtihin the Docker container.

Optional `environment`, a map of encironment variable names and values.

#### `branches`
Defines rules for whitelisting/blacklisting eecution of some branches if Workflows are not configured. If you are using Workflows, job-level branches will be ignored and must be configured in the Workflows section of your `config.yml` file.

```YAML
jobs:
  build:
    branches:
      only:
        - master
        - /rc-.*/
```

```YAML
jobs:
  build:
    branches:
      ignore:
        - develop
        - /feature-.*/
```

If both `ignore` and `only` are present, only `ignore` will be taken into account.

#### `steps`
Should be a list of single key/value pairs, the key of which indicates the step type. The value may be either a configuration map or a string.

```YAML
jobs:
  build:
    working_directory: ~/canary-python
    environment:
      FOO: bar
    steps:
      - run:
          name: Running tests # used in the UI for display purposes
          command: make test # defines command to execute
```

Short form:
```YAML
jobs:
  build:
    steps:
      - run: make test
```

The `checkout` step will checkout project source code into the job's `working_directory`.

#### Built-in steps:
**`run`**
* Used for invoking all command-line programs, taking either a map of config values, or a string. Run commands are executed using non-login shells by default, so you must explicitly source any dotfiles as part of the command.

* `command` is a required String, which is the command to run via the shell.

* `name` is an optional String, the title of the step to be shown in the CircleCI UI.

* `shell` is an optional String, to define the shell for execution of the command.

* `environment` is an optional String, for defining additional environment variables.

* `when` is an optional String, specifying when to enable or disable the step. Takes `always`, `on_success`, and `on_fail`.

To specify a multi-line `command`, each line of which will be run in the same shell:
```YAML
- run:
    command: |
      echo Running test
      mkdir -p /tmp/test-results
      make test
```

**`checkout`**
* Special step used to check out source code to the configured `path` (defaults to the `working_directory`). This step configures git to checkout over ssh.

**`save_cache`**
* Generates and stores a cache of a file or directory of files such as dependencies or source code in our objct storage. Later jobs can restore this cache. See [caching documentation](https://circleci.com/docs/2.0/caching/).

* `paths` is a required List, of directories which should be added to the cache.

* `key` is a required String, a unique identifier for this cache.

The cache for a specific `key` is immutable and cannot be changed once written.

Template examples:
* `myapp-{{ checksum "package.json" }}`

* `myapp-{{ .Branch }}-{{ checksum "package.json" }}`

**`restore_cache`**
* Restores a previously saved cache based on a `key`.

* `key` is a required String, a single cache key to restore.

* `keys` is alternate to required `key`, is a List of cache keys to lookup for a cache to restore. Only first existing key will be restored.

**`deploy`**
* Special step for deploying artifacts.

* Uses the same configuration map and semantics as `run` step. Jobs may have more than one `deploy` step.

Example:
```YAML
- deploy:
    command: |
      if [ "${CIRCLE_BRANCH}" == "master" ]; then
        ansible-playbook site.yml
      fi
```

**`add_ssh_keys`**
* Special step that adds SSH keys from a project's settings to a container. Also configures SSH to use these keys.

* `fingerprints` is an optional List of strings corresponding to the keys to be added.

* **Note:** Even though CircleCI uses `ssh-agent` to sign all added SSH keys, you **must** use the `add_ssh_keys` key to actually add keys to a container.

**`store_artifacts`**
* Step to store artifacts (for example logs, binaries, etc) to be available in the web app or through API.

* `path` is a required String, the directory in the primary container to save as job artifacts

* `destination` is optional string prefix added to artifact paths in artifacts API.

```YAML
- store_artifacts:
    path: /code/test-results
    destination: prefix
```

**`store_test_results`**
* Special step used to upload test results to display in builds' Test Summary section.

* `path` is a required String, path to directory containing subdirectories of JUnit XML or Cucumber JSON test metadata files.

**`persist_to_workspace`**
* Special step used to persist a temporary file to be used by another job in the workflow.

* `root` is a required String, either an absolute path or path relative to `working_directory`

* `paths` is a required List, Glob identifying file(s) or non-glob path to a directory to add to the shared workspace.

The root key is a directory on the container which is taken to be the root directory of the workspace. The paths values are all relative to the root.

**Example:** The following step syntax persists the specified paths from `/tmp/dir` into the workspace, relative to the directory `
/tmp/dir`.

```YAML
persist_to_workspace:
  root: /tmp/dir
  paths:
    - foo/bar
    - baz
```

After this step completes, the folloing directories are added to the workspace:

```
foo/bar
baz
```

**`attach_workspace`**
* Special step used to attach the workflow's workspace to the current container. The full contents of the workspace are downloaded and copied into the directory the workspace is being attached at.

* `at` is a required String, directory to attach the workspace to.

```YAML
- attach_workspace:
    at: /tmp/workspace
```

Each workflow has a temporary workspace associated with it. The workspace can be used to pass along unique data built during a job to other jobs in the same workflow. Jobs can add files into the workspace using the `persist_to_workspace` step and download the workspace content into their filesystem using the `attach_workspace` step. The workspace is additive only, jobs may add files to the workspace but cannot delete files from the workspace. Each job can only see content added to the workspace by the jobs that are upstream of it.



## `workflows`
Used for orchestrating all jobs. Each workflow consists of the workflow name as a key and a map as a value. A name should be unique within the current `config.yml`. The top-level keys for the Workflows configuration are `version` and `jobs`.

### `version`
Should be 2.

### &lt;`workflow_name`&gt;
A unique name for your workflow.

#### `triggers`
Specifies which triggers will cause this workflow to be executed. Default behavior is to trigger the workflow when **pushing to a branch**.

**`schedule`**
* A workflow may have a `schedule` indicating it runs at a certain time. For example, a nightly build that runs every day at 12am UTC:
```YAML
workflows:
   version: 2
   nightly:
     triggers:
       - schedule:
           cron: "0 0 * * *" # POSIX crontab syntax
           filters:
             branches:
               only:
                 - master
                 - beta
     jobs:
       - test
```

### jobs
A job can have the keys `requires`, `filters`, and `context`.

`jobs` is a required List of jobs to run with their dependencies.

#### &lt;`job_name`&gt;
A job name that exists in your `config.yml`.

**`requires`**
* Jobs are run in parallel by default, so you must explicitly require any dependencies by their job name.

* `requires` is a List of jobs that must succeed for the job to start.

**`context`**
* Jobs may be configured to use global environment variables set for an organization. 

* `context` is a String of the name of the context. The initial default is `org-global`.

**`type`**
* A job may have a `type` of `approval` indicating it must be manually approved before downstream jobs may proceed. Jobs run in the dependency order until the workflow processes a job with the `type: approval` key followed by a job in which it depends, for example:

```YAML
- hold:
    type: approval
    requires:
    - test1
    - test2
- deploy:
    requires:
    - hold
```

**`branches`**
* Same as branches for jobs.

## Full Example
```YAML
version: 2
jobs:
    build:
        docker:
            - image: ubuntu:14.04

            - image: mongo:2.6.8
            command: [mongod, --smallfiles]
            
            - image: postgres:9.4.1
            # some containers require setting environment variables
            environment:
                POSTGRES_USER: root
            
            - image: rabbitmq:3.5.4

        environment:
            TEST_REPORTS: /tmp/test-reports

        working_directory: ~/my-project

        branches:
            ignore:
                - develop
                - /feature-.*/

        steps:
            - checkout

            - run:
                command: echo 127.0.0.1 devhost | sudo tee -a /etc/hosts

            # Create Postgres users and database
            # Note the yaml heredoc '|' for nicer formatting
            - run: |
                sudo -u root createuser -h localhost --superuser ubuntu &&
                sudo createdb -h localhost test_db
            
            - restore_cache:
                keys:
                    - v1-my-project-{{ checksum "project.clj" }}
                    - v1-my-project-

            - run:
                environment:
                    SSH_TARGET: "localhost"
                    TEST_ENV: "linux"
                command: |
                    set -xu
                    mkdir -p ${TEST_REPORTS}
                    run-tests.sh
                    cp out/tests/*.xml ${TEST_REPORTS}

            - run: |
                set -xu
                mkdir -p /tmp/artifacts
                create_jars.sh ${CIRCLE_BUILD_NUM}
                cp *.jar /tmp/artifacts

            - save_cache:
                key: v1-my-project-{{ checksum "project.clj" }}
                paths:
                    - ~/.m2

            # Save artifacts
            - store_artifacts:
                path: /tmp/artifacts
                destination: build
            
            # Upload test results
            - store_test_results:
                path: /tmp/test-reports

    deploy-stage:
        docker:
            - image: ubuntu:14.04
        working_directory: /tmp/my-project
        steps:
            - run:
                name: Deploy if tests pass and branch is Staging
                command: ansible-playbook site.yml -i staging

    deploy-prod:
        docker:
            - image: ubuntu:14.04
        working_directory: /tmp/my-project
        steps:
            - run:
                name: Deploy if tests pass and branch is Master
                command: ansible-playbook site.yml -i production

workflows:
    version: 2
    build-deploy:
        jobs:
            - build
            - deploy-stage:
                requires:
                    -build
                filters:
                    branches:
                        only: staging
            - deploy-prod:
                requires:
                    -build
                filters:
                    branches:
                        only: master
```