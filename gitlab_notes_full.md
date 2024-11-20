
# GitLab CI/CD Notes

## General Notes

- **`. ->`** Add a dot before a job's name to hide it from the pipeline.
- **`image: ubuntu`** Specifies the base image used to run the pipeline.
- **`before_script:` / `after_script:`** Defines commands to execute before or after the main script.
    ```yaml
    before_script:
      - apt update; apt install -y gcc
    after_script:
      - echo "Finished!"
    ```

## Cache Configuration

Defines paths to cache for reuse in subsequent jobs.

```yaml
cache:
  paths:
    - "./program"  # Specifies what to save for other jobs
    - "./test"     # Additional path
```

## Stages

Stages define the order in which jobs are executed.

```yaml
stages:
  - syntax
  - compile
  - test
  - deploy
```

## Example Jobs

### Syntax Check

```yaml
check:
  image: gcc
  stage: syntax
  script:
    - gcc -fsyntax-only program.c
```

### Compilation Job

```yaml
compiling:
  image: gcc
  stage: compile
  only:
    - test  # Only runs on the "test" branch
  script:
    - gcc -v -o program program.c 2> gcc.log  # Logs compilation output
  artifacts:
    paths:
      - "gcc.log"  # Save compilation logs as artifacts
    expire_in: 1 week
    when: on_failure  # Save artifacts only on failure
```

### Testing Job

```yaml
testing:
  stage: test
  script:
    - bash test.sh
```

### Cleanup Job

```yaml
cleaning:
  stage: test
  script:
    - rm ./program
  when: on_failure  # Run cleanup job on failure of previous jobs
  start_in: 30 minutes  # Delay execution
  except:
    - main  # Skip this job on the "main" branch
```

### Deployment Job

```yaml
upload:
  stage: deploy
  script:
    - echo "Uploading app..."
  when: manual  # Requires manual execution
```

## Job Dependencies with `needs`

```yaml
execute_A:
  needs: [compile_A]  # Waits for "compile_A" job to finish
  stage: execute
  script:
    - echo "Running the program..."
    - java HelloWorld
```

## SSH in GitLab CI

```yaml
script:
  - apk update; apk add openssh-client
  - eval $(ssh-agent -s)
  - echo "$SSH_PRIVATE_KEY" | tr -d '' | ssh-add - > /dev/null
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - ssh-keyscan $AWS_IP >> ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts
  - ssh $USER@$AWS_IP "date;"
```

## GitLab Pages

```yaml
pages:
  image: alpine:latest
  script:
    - mkdir -p ./public
    - cp ./*.html ./public/
  artifacts:
    paths:
      - public  # Content for the GitLab Pages site
  except:
    - main
```

## Environments

```yaml
job_branch:
  stage: deploy
  except:
    - main
  script:
    - echo "data na $CI_ENVIRONMENT_SLUG" >> public/index.html
  environment:
    name: staging/$CI_COMMIT_REF_NAME
    url: $CI_PAGES_URL

job_master:
  stage: deploy
  only:
    - main
  when: manual
  script:
    - echo "data na produkcii" >> public/index.html
  environment:
    name: production
    url: $CI_PAGES_URL
```

## Variables

### Global and Job-Specific Variables

```yaml
variables:
  VAR: "value"  # Global variable

test-variable:
  variables:
    VAR2: "value2"  # Job-specific variable
  stage: test
  script:
    - echo $VAR2
```

### Common GitLab CI/CD Variables

#### General Pipeline and Job Variables

- **`$CI_PIPELINE_ID`**: Unique ID of the pipeline.
- **`$CI_JOB_ID`**: Unique ID of the job.
- **`$CI_JOB_NAME`**: Name of the job.
- **`$CI_JOB_STAGE`**: Stage of the current job.
- **`$CI_JOB_STATUS`**: Status of the current job (e.g., success, failed).
- **`$CI_COMMIT_SHA`**: SHA hash of the commit being built.
- **`$CI_COMMIT_SHORT_SHA`**: Shortened version of the commit SHA.
- **`$CI_COMMIT_REF_NAME`**: Branch or tag name for the commit.
- **`$CI_COMMIT_MESSAGE`**: Full commit message.

#### Project and Repository Variables

- **`$CI_PROJECT_ID`**: Unique ID of the project.
- **`$CI_PROJECT_NAME`**: Name of the project.
- **`$CI_PROJECT_PATH`**: Path to the project, including namespace (e.g., username/project-name).
- **`$CI_PROJECT_NAMESPACE`**: Namespace or group containing the project.
- **`$CI_PROJECT_URL`**: URL to the GitLab project.
- **`$CI_REPOSITORY_URL`**: URL to the Git repository.

#### Environment and Deployment Variables

- **`$CI_ENVIRONMENT_NAME`**: Name of the environment.
- **`$CI_ENVIRONMENT_SLUG`**: URL-safe version of the environment name.
- **`$CI_ENVIRONMENT_URL`**: URL of the environment.
- **`$CI_ENVIRONMENT_ACTION`**: The action to be performed on the environment (start, stop, prepare).

#### User and Runner Information

- **`$CI_RUNNER_ID`**: ID of the GitLab Runner used for the job.
- **`$CI_RUNNER_DESCRIPTION`**: Description of the GitLab Runner.
- **`$CI_RUNNER_TAGS`**: Tags associated with the GitLab Runner.
- **`$GITLAB_USER_EMAIL`**: Email of the GitLab user who triggered the pipeline.
- **`$GITLAB_USER_LOGIN`**: Username of the GitLab user who triggered the pipeline.

#### GitLab Pages Specific Variables

- **`$CI_PAGES_URL`**: URL of the GitLab Pages site (e.g., https://namespace.gitlab.io/project).

#### Debugging and Logging Variables

- **`$CI_DEBUG_TRACE`**: If set to true, enables debug logging for the job.

## Pipeline Trigger

```bash
curl -X POST      --fail      -F token=glptt-ff022b454b22faefffdf14861a30a6dba6884453      -F "ref=main" \  # Branch to trigger
     -F "variables[VAR2]=VALUE" \  # Pass variables
     https://gitlab.com/api/v4/projects/64280095/trigger/pipeline
```

## Container Registry

```yaml
script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  - docker build -t registry.gitlab.com/marvy936/lab14 .
  - docker push registry.gitlab.com/marvy936/lab14
```

## Releases

```yaml
image: registry.gitlab.com/gitlab-org/release-cli:latest
stages:
  - compile
  - storage
  - release

ios-release:
  stage: release
  script:
    - echo 'iOS release job $CI_COMMIT_SHORT_SHA'
  rules:
    - if: $RELEASE == 'true'
  release:
    name: 'Release iOS $CI_COMMIT_SHORT_SHA'
    tag_name: '$CI_COMMIT_SHORT_SHA'
    description: 'iOS release v1.0.0 $CI_COMMIT_SHORT_SHA'
    assets:
      links:
        - name: 'iOS'
          url: 'http://$CI_PAGES_URL/$CI_PROJECT_NAME/program'
```

## REST API

```bash
curl --header "PRIVATE-TOKEN: {access_token}" "https://gitlab.example.com/api/v4/projects/{project_id}/pipelines"
```

- Official documentation: [GitLab REST API](https://docs.gitlab.com/ee/api/rest/)
