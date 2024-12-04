# Gitlab Notes

## Table of Contents

1. [GitLab Pipeline `.gitlab-ci.yml`](#gitlab-pipeline-.gitlab-ci.-yml)
2. [Additional Notes](#additional-notes)
    - [Hidden Jobs](#hidden-jobs)
3. [Environments](#environments)
4. [SSH Setup in GitLab](#ssh-setup-in-gitlab)
5. [Using the `needs` Keyword](#using-the-needs-keyword)
6. [GitLab Pages Configuration](#gitlab-pages-configuration)
7. [Variables](#variables)
    - [Full List of Built-In Variables](#full-list-of-built-in-variables)
8. [Pipeline Trigger](#pipeline-trigger)
9. [Container Registry](#container-registry)
10. [Releases](#releases)
11. [Package Registry](#package-registry)

---

# GitLab Pipeline `.gitlab-ci.yml`

```yaml
# Base image to use in all jobs unless overridden in specific jobs.
image: ubuntu 

# Commands to run before each job's main script section.
before_script: 
  # Updates package lists and installs gcc in the job container.
  - apt update 
  - apt install -y gcc

# Caches files or directories between pipeline runs.
cache:
  paths:
    # Example paths to cache. Cached files can be reused by other jobs.
    - "./program"
    - "./test"

# Defines the sequence of stages in the pipeline and their execution order.
stages: 
  - syntax     # First stage: Syntax checks.
  - compile    # Second stage: Compilation.
  - test       # Third stage: Testing.
  - deploy     # Fourth stage: Deployment.

# Example syntax check job.
check: 
  # Specific base image for this job (overrides the global image).
  image: gcc 
  # Associates the job with the "syntax" stage.
  stage: syntax 
  # Commands to execute for this job.
  script: 
    # Perform a syntax-only check on program.c using GCC.
    - gcc -fsyntax-only program.c

# Example compile job.
compiling:
  image: gcc           # Use GCC image for this job.
  stage: compile       # This job belongs to the "compile" stage.
  only:                # This job will only run on the "test" branch.
    - test
  script:
    # Compile the program and redirect errors to gcc.log.
    - gcc -v -o program program.c 2> gcc.log 
  artifacts:           # Saves the gcc.log file as an artifact.
    paths:
      - "gcc.log"      # File path to save as an artifact.
    expire_in: 1 week  # Artifacts will be stored for one week.
    when: on_failure   # Save artifacts only if the job fails.

# Example testing job.
testing: 
  stage: test          # This job belongs to the "test" stage.
  script:
    # Runs a test script named test.sh.
    - bash test.sh

# Cleanup task after testing.
cleaning:
  stage: test          # Part of the "test" stage.
  script:
    # Removes the program binary after testing.
    - rm ./program
  when: on_failure     # Runs only if the previous job fails.
  except: 
    # Skip this job on the main branch.
    - main

# Example deployment job.
upload:
  stage: deploy        # This job belongs to the "deploy" stage.
  script:
    # Outputs a message to indicate deployment.
    - echo "uploading app..."
  when: manual         # Requires manual trigger to execute.
```

---

# Additional Notes

### Hidden Jobs
```yaml
. -> add dot before job's name to hide it from pipeline.
```
- Jobs starting with a `.` are hidden by default and won’t appear in the pipeline view. These can still be referenced by other jobs (e.g., via `extends`).

---

# Environments
Environments in GitLab allow the specification of deployment targets like staging and production.

```yaml
job_branch:
  stage: deploy                    # Assign this job to the "deploy" stage.
  except:                          
    - main                         # Exclude the "main" branch; this job won't run on it.
  script:
    # This script is executed as part of the deployment process for the staging environment.
    - echo "Deploying to staging environment"
  environment:                     # Define the environment details for this deployment.
    name: staging/$CI_COMMIT_REF_NAME
    url: $CI_PAGES_URL

    # 'name' specifies the environment. The use of $CI_COMMIT_REF_NAME ensures
    # that each branch has its own staging environment (e.g., 'staging/feature-branch').
    
    # 'url' specifies the URL associated with this environment.
    # $CI_PAGES_URL is a built-in GitLab variable that provides the GitLab Pages URL for the project.
```

---

# SSH Setup in GitLab
SSH setup allows secure remote access during CI/CD jobs.

```yaml
script:
  - apk update; apk add openssh-client                            # Installs the SSH client on the job's container.
  - eval $(ssh-agent -s)                                          # Starts the SSH agent for managing keys.
  - echo "$SSH_PRIVATE_KEY" | tr -d '\\r' | ssh-add - > /dev/null # Adds the private key to the SSH agent. The `$SSH_PRIVATE_KEY` is a variable
                                                                  # that must be set in **Settings > CI/CD > Variables**.
                                                                  # `tr -d '\\r'` ensures compatibility with UNIX line endings.
  - mkdir -p ~/.ssh                                               # Creates the SSH configuration directory if it doesn't exist.
  - chmod 700 ~/.ssh                                              # Secures the `.ssh` directory.
  - ssh-keyscan $AWS_IP >> ~/.ssh/known_hosts                     # Adds the remote server's public key to the known_hosts file.
                                                                  # This prevents "unknown host" errors during SSH connections.
                                                                  # `$AWS_IP` should be defined in your CI/CD variables.
  - chmod 644 ~/.ssh/known_hosts                                  # Adjusts permissions for the known_hosts file.
  - ssh $USER@$AWS_IP "date;"                                     # Tests the connection by running the `date` command on the remote server.
```
Key Points:

    Environment Variables:
        $SSH_PRIVATE_KEY: The private SSH key for authentication. Add this as a protected variable in your GitLab project.
        $AWS_IP: The remote server's IP address. Add this as a variable in Settings > CI/CD > Variables.

    Security Best Practices:
        Use protected and masked variables for sensitive data like SSH keys.
        Ensure the private key has the correct permissions (chmod 600).

    Common Use Cases:
        Deploying files to a server using scp.
        Running deployment scripts on a remote server.
---

# Using the `needs` Keyword
The `needs` keyword defines specific dependencies between jobs.

```yaml
execute_A:
  needs: [compile_A]                    # Specifies that this job depends on the "compile_A" job.
  tags:
    - telekom                           # Uses a runner tagged with "telekom".
  stage: execute                        # Assigns this job to the "execute" stage.
  script:
    - echo "Running the program..."     # Prints a message to indicate execution.
    - java HelloWorld                   # Runs the HelloWorld program.
```
Key Points:

    Parallel Execution:
        Jobs with needs can run in parallel with other jobs as long as their dependencies are met.
        This speeds up pipelines by avoiding stage-level waiting.

    Syntax:
        needs: [job_name]: Lists the job(s) this job depends on.
        Dependencies can span across different stages.
        
---

# GitLab Pages Configuration
```yaml
pages:
  image: alpine:latest        # Lightweight image for Pages jobs.
  script:
    # Create the public directory (required for GitLab Pages).
    - mkdir -p ./public
    # Copy all .html files to the public directory, which GitLab Pages serves.
    - cp ./*.html ./public/
  artifacts:
    paths:
      - public               # Save the "public" folder as an artifact.
  except:
    - main                   # Skip this job on the main branch.
```
- **Purpose**: Sets up GitLab Pages to host static files.
- **Artifacts**: Files in the `public` directory are required for GitLab Pages to serve them.
- **Branch Restriction**: Skips execution on the `main` branch.

---

# Variables
```yaml
variables:
  VAR: "value"               # Example of a global variable.

test-variable:
  variables:
    VAR2: "value2"           # Example of a job-specific variable.
  stage: test
  script:
    # Outputs both global and job-specific variables.
    - echo $VAR
    - echo $VAR2
```
- **Global Variables**: Accessible across all jobs and stages.
- **Job-Specific Variables**: Defined within a job and accessible only there.
- **Built-In Variables**: GitLab provides many predefined variables, such as:
  - `$CI_COMMIT_REF_NAME`: Name of the branch or tag.
  - `$CI_PIPELINE_ID`: Unique pipeline ID.
  - `$CI_JOB_NAME`: Name of the current job.

# Full List of Built-In Variables

General Pipeline and Job Variables

    $CI_PIPELINE_ID: Unique ID of the pipeline.
    $CI_JOB_ID: Unique ID of the job.
    $CI_JOB_NAME: Name of the job.
    $CI_JOB_STAGE: Stage of the current job.
    $CI_JOB_STATUS: Status of the current job (e.g., success, failed).
    $CI_COMMIT_SHA: SHA hash of the commit being built.
    $CI_COMMIT_SHORT_SHA: Shortened version of the commit SHA.
    $CI_COMMIT_REF_NAME: Branch or tag name for the commit.
    $CI_COMMIT_MESSAGE: Full commit message.

Project and Repository Variables

    $CI_PROJECT_ID: Unique ID of the project.
    $CI_PROJECT_NAME: Name of the project.
    $CI_PROJECT_PATH: Path to the project, including namespace (e.g., username/project-name).
    $CI_PROJECT_NAMESPACE: Namespace or group containing the project.
    $CI_PROJECT_URL: URL to the GitLab project.
    $CI_REPOSITORY_URL: URL to the Git repository.

Environment and Deployment Variables

    $CI_ENVIRONMENT_NAME: Name of the environment defined in the .gitlab-ci.yml file.
    $CI_ENVIRONMENT_SLUG: URL-safe version of the environment name.
    $CI_ENVIRONMENT_URL: URL of the environment (useful for GitLab Pages).
    $CI_ENVIRONMENT_ACTION: The action to be performed on the environment (start, stop, or prepare).

User and Runner Information

    $CI_RUNNER_ID: ID of the GitLab Runner used for the job.
    $CI_RUNNER_DESCRIPTION: Description of the GitLab Runner.
    $CI_RUNNER_TAGS: Tags associated with the GitLab Runner.
    $GITLAB_USER_EMAIL: Email of the GitLab user who triggered the pipeline.
    $GITLAB_USER_LOGIN: Username of the GitLab user who triggered the pipeline.

GitLab Pages Specific Variables

    $CI_PAGES_URL: URL of the GitLab Pages site (e.g., https://namespace.gitlab.io/project).

Debugging and Logging Variables

    $CI_DEBUG_TRACE: If set to true, enables debug logging for the job.
    $CI_SERVER_NAME: Name of the GitLab instance.

Customization Variables

    $CI_COMMIT_REF_SLUG: URL-safe version of the branch or tag name.
    $CI_COMMIT_REF_PROTECTED: true if the branch or tag is protected, otherwise false.
---

# Pipeline Trigger

To set up a pipeline trigger in GitLab, follow these steps:

    Navigate to your project in GitLab.
    Go to Settings > CI/CD > Pipeline Triggers.
    Add a new token by clicking Add trigger.
    A unique token will be generated. Beneath the token, you’ll see a button 
    labeled View trigger token usage examples. This provides a pre-built curl command for triggering pipelines.
    
```bash
curl -X POST \
     --fail \
     -F token=glptt-ff022b454b22faefffdf14861a30a6dba6884453 \
     -F "ref=main" \             # Specifies the branch to trigger.
     -F "variables[VAR2]=VALUE" \ # Pass variables to the pipeline.
     https://gitlab.com/api/v4/projects/64280095/trigger/pipeline
```

What does each part do?

    -X POST: Sends a POST request to trigger the pipeline.
    --fail: Ensures the script fails if the trigger is unsuccessful.
    -F token=<your-token>: Includes the unique trigger token.
    -F "ref=<branch>": Specifies the branch or tag to build.
    -F "variables[<variable-name>]=<value>": Injects custom variables into the pipeline.
    <GitLab API URL>: The endpoint to which the POST request is sent.

Using this setup, you can programmatically start your pipelines and customize their behavior with variables. Let me know if you'd like detailed comments added to another section!

---

# Container Registry
```yaml
script: 
  # Authenticate with the GitLab Container Registry.
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  # Build the Docker image.
  - docker build -t registry.gitlab.com/user/project . 
  # Push the image to the registry.
  - docker push registry.gitlab.com/user/project
```
- **Purpose**: Builds and pushes Docker images to the GitLab Container Registry.
- **Environment Variables**:
  - `$CI_REGISTRY`: Registry URL (e.g., `registry.gitlab.com`).
  - `$CI_REGISTRY_USER`: Username for the registry.
  - `$CI_REGISTRY_PASSWORD`: Password/token for authentication.

Store sensitive information like CI_REGISTRY_USER and CI_REGISTRY_PASSWORD in GitLab’s CI/CD Variables. To set this up:

    Go to your project in GitLab.
    Navigate to Settings > CI/CD > Variables.
    Add your variables:
        CI_REGISTRY
        CI_REGISTRY_USER
        CI_REGISTRY_PASSWORD

These variables will automatically be available in your pipeline jobs and are securely masked in logs.
Or you can EXPORT them in script:

```
script:
  - export CI_REGISTRY="registry.gitlab.com"
  - export CI_REGISTRY_USER="USER"
  - export CI_REGISTRY_PASSWORD="PASS/TOKEN"
  - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
```

---

# Releases
```yaml
ios-release:
  image: registry.gitlab.com/gitlab-org/release-cli:latest  # Required release image.
  stage: release
  script:
    # Outputs information about the release.
    - echo 'iOS release job $CI_COMMIT_SHORT_SHA'
  rules:
    - if: $RELEASE == 'true'   # Only runs if the RELEASE variable is true.
  release:
    name: 'Release iOS $CI_COMMIT_SHORT_SHA'   # Name of the release.
    tag_name: '$CI_COMMIT_SHORT_SHA'           # Tag for the release.
    description: 'iOS release v1.0.0 $CI_COMMIT_SHORT_SHA' # Details.
    assets:
      links:
        - name: 'iOS'        # Friendly name for the asset.
          url: 'http://$CI_PAGES_URL/$CI_PROJECT_NAME/program' # URL to the asset.
```
- **Purpose**: Automates creating a GitLab release.
- **Tag Name**: Each release requires a unique Git tag.
- **Assets**: Links associated with the release, such as build artifacts or documentation.

---

# Package Registry
```bash
curl --header "JOB-TOKEN: $CI_JOB_TOKEN" --upload-file {path_to_file} "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/{package_name}/1.0.1/HelloWorld.class"

wget --header="JOB-TOKEN: $CI_JOB_TOKEN" ${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/generic/my_package/1.0.1/HelloWorld.class

$CI_API_V4_URL = https://gitlab.com/api/v4
```
