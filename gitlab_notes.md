# GitLab Notes

## Table of Contents

1. [Basic Job Configuration](#basic-job-configuration)
2. [GitLab Needs](#gitlab-needs)
3. [GitLab SSH](#gitlab-ssh)
4. [GitLab Pages](#gitlab-pages)
5. [GitLab Environments](#gitlab-environments)
6. [GitLab Variables](#gitlab-variables)
   - [General Pipeline and Job Variables](#general-pipeline-and-job-variables)
   - [Project and Repository Variables](#project-and-repository-variables)
   - [Environment and Deployment Variables](#environment-and-deployment-variables)
   - [User and Runner Information](#user-and-runner-information)
   - [GitLab Pages Specific Variables](#gitlab-pages-specific-variables)
   - [Debugging and Logging Variables](#debugging-and-logging-variables)
   - [Customization Variables](#customization-variables)
7. [Pipeline Trigger](#pipeline-trigger)
8. [Container Registry](#container-registry)
9. [Releases](#releases)
10. [Package Registry](#package-registry)
11. [REST API](#rest-api)

---

## Basic Job Configuration

- **Hide a Job**: Add a `.` before the job's name to hide it from the pipeline.
- **Image**: Base image used for the pipeline.
  ```yaml
  image: ubuntu
  ```
- **Before/After Scripts**: Commands to run before or after the job.
  ```yaml
  before_script:
    - apt update; apt install -y gcc
  after_script:
    - echo "Cleanup done"
  ```
- **Cache**: Specify paths to cache for other jobs.
  ```yaml
  cache:
    paths:
      - "./program"
      - "./test"
  ```
- **Stages**: Define stages in the pipeline in order of execution.
  ```yaml
  stages:
    - syntax
    - compile
    - test
    - deploy
  ```

### Example Job Configuration
```yaml
check:
  image: gcc
  stage: syntax
  script:
    - gcc -fsyntax-only program.c
```

---

## GitLab Needs
```yaml
execute_A:
  needs: [compile_A]  # Waits for the job to finish, not the whole stage.
  tags:
    - telekom
  stage: execute
  script:
    - echo "Running the program..."
    - java HelloWorld
```

---

## GitLab SSH
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

## GitLab Pages
- Links to pages are available in **Deploy > Pages**.
- Visibility can be changed in **Settings > Visibility > Pages**.

```yaml
pages:
  image: alpine:latest
  script:
    - mkdir -p ./public
    - cp ./*.html ./public/
  artifacts:
    paths:
      - public
  except:
    - main
```

---

## GitLab Environments
```yaml
job_branch:
  stage: deploy
  except:
    - main
  script:
    - echo "Data on $CI_ENVIRONMENT_SLUG" >> public/index.html
  environment:
    name: staging/$CI_COMMIT_REF_NAME
    url: $CI_PAGES_URL

job_master:
  stage: deploy
  only:
    - main
  when: manual
  script:
    - echo "Production data" >> public/index.html
  environment:
    name: production
    url: $CI_PAGES_URL
```

---

## GitLab Variables

### General Pipeline and Job Variables
- `$CI_PIPELINE_ID`: Unique ID of the pipeline.
- `$CI_JOB_ID`: Unique ID of the job.
- `$CI_JOB_NAME`: Name of the job.
- `$CI_COMMIT_SHA`: SHA hash of the commit being built.

### Project and Repository Variables
- `$CI_PROJECT_ID`: Unique ID of the project.
- `$CI_PROJECT_NAME`: Name of the project.

### Environment and Deployment Variables
- `$CI_ENVIRONMENT_NAME`: Name of the environment.

### User and Runner Information
- `$CI_RUNNER_ID`: ID of the GitLab Runner.

### GitLab Pages Specific Variables
- `$CI_PAGES_URL`: URL of the GitLab Pages site.

---

## Pipeline Trigger
```bash
curl -X POST      --fail      -F token={trigger_token}      -F "ref=main"      -F "variables[VAR]=VALUE"      https://gitlab.com/api/v4/projects/{project_id}/trigger/pipeline
```

---

## Container Registry
```yaml
script:
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  - docker build -t registry.gitlab.com/username/project .
  - docker push registry.gitlab.com/username/project
```

---

## Releases
```yaml
image: registry.gitlab.com/gitlab-org/release-cli:latest
stages:
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

---

## Package Registry
```bash
curl --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file {path_to_file} "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/{package_name}/1.0.1/HelloWorld.class"

wget --header="JOB-TOKEN: $CI_JOB_TOKEN" ${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/my_package/1.0.1/HelloWorld.class
```

---

## REST API
```bash
curl --header "PRIVATE-TOKEN: {access_token}" "https://gitlab.example.com/api/v4/projects/{project_id}/pipelines"
```

---

