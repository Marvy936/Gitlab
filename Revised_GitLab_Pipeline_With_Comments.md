
# GitLab CI/CD Pipeline Documentation with Comments

This document covers GitLab CI/CD pipeline features, including environments, SSH setup, the `needs` keyword, REST API triggers, and a comprehensive list of built-in variables.

---

## GitLab Pipeline Example

```yaml
# This pipeline demonstrates various GitLab CI/CD configurations and features.

stages:
  - syntax
  - compile
  - test
  - deploy
  - cleanup

.syntax_check:
  image: gcc
  stage: syntax
  script:
    - gcc -fsyntax-only program.c
  cache:
    paths:
      - "./program"
      - "./test"

compiling:
  image: gcc
  stage: compile
  only:
    - test
  script:
    - gcc -v -o program program.c 2> gcc.log
  artifacts:
    paths:
      - gcc.log
    expire_in: 1 week
    when: on_failure

testing:
  stage: test
  script:
    - bash test.sh

cleanup:
  stage: cleanup
  script:
    - rm ./program
  when: always

deploy_staging:
  stage: deploy
  except:
    - main
  script:
    - echo "Deploying to staging environment"
  environment:
    name: staging/$CI_COMMIT_REF_NAME
    url: $CI_PAGES_URL

deploy_production:
  stage: deploy
  only:
    - main
  when: manual
  script:
    - echo "Deploying to production"
  environment:
    name: production
    url: $CI_PAGES_URL
```

---

## Additional Features

### Environments
Environments in GitLab allow the specification of deployment targets like staging and production.

```yaml
job_branch:
  stage: deploy
  except:
    - main
  script:
    - echo "Deploying to staging environment"
  environment:
    name: staging/$CI_COMMIT_REF_NAME
    url: $CI_PAGES_URL
```

---

### SSH Setup in GitLab
SSH setup allows secure remote access during CI/CD jobs.

```yaml
script:
  - apk update; apk add openssh-client
  - eval $(ssh-agent -s)
  - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - ssh-keyscan $AWS_IP >> ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts
  - ssh $USER@$AWS_IP "date;"
```

---

### Using the `needs` Keyword
The `needs` keyword defines specific dependencies between jobs.

```yaml
execute_A:
  needs: [compile_A]
  tags:
    - telekom
  stage: execute
  script:
    - echo "Running the program..."
    - java HelloWorld
```

---

### REST API Pipeline Trigger
Programmatically start pipelines and pass variables using the REST API.

#### Example Trigger
1. Navigate to **Settings > CI/CD > Pipeline triggers** in your GitLab project.
2. Add a new trigger and use the generated token.

```bash
curl -X POST      --fail      -F token=YOUR_TRIGGER_TOKEN      -F "ref=main"      -F "variables[MY_VAR]=my_value"      https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/trigger/pipeline
```

---

### Full List of Built-In Variables

#### General Pipeline and Job Variables
- `$CI_PIPELINE_ID`: Unique ID of the pipeline.
- `$CI_JOB_ID`: Unique ID of the job.
- `$CI_JOB_NAME`: Name of the job.
- `$CI_JOB_STAGE`: Stage of the current job.
- `$CI_COMMIT_SHA`: Full commit SHA hash.

#### Project and Repository Variables
- `$CI_PROJECT_ID`: Unique ID of the project.
- `$CI_PROJECT_NAME`: Name of the project.
- `$CI_REPOSITORY_URL`: URL to the Git repository.

#### Environment and Deployment Variables
- `$CI_ENVIRONMENT_NAME`: Name of the environment.
- `$CI_ENVIRONMENT_SLUG`: URL-safe version of the environment name.
- `$CI_ENVIRONMENT_URL`: URL of the environment.

#### User and Runner Information
- `$CI_RUNNER_ID`: ID of the GitLab Runner.
- `$CI_RUNNER_DESCRIPTION`: Description of the GitLab Runner.

#### GitLab Pages Variables
- `$CI_PAGES_URL`: URL of the GitLab Pages site.

#### Debugging and Logging Variables
- `$CI_DEBUG_TRACE`: If set to true, enables debug logging for the job.
