# AGENTS.md: uploading test results to Allure TestOps with allurectl

This file tells AI coding agents how to use `allurectl` to upload Allure test results to Allure TestOps from a local machine and from CI/CD pipelines.

`allurectl` is a closed-source command-line tool. The only sources of truth are the built-in help (`allurectl --help`, `allurectl <command> --help`) and the public documentation at https://docs.qameta.io/allure-testops/reference/ecosystem/allurectl/. Do not guess flags, variables or behaviour that are not in one of these two places. If the installed version prints a different flag list than this file, trust `--help`.

## Rules to follow before anything else

1. One CI build gives one launch. If the task is to get several launches out of a single CI build, stop and send the user to support: https://help.qameta.io. See [Several launches from one CI build](#several-launches-from-one-ci-build).
2. Check whether the CI system is in the [natively supported list](#natively-supported-ci-systems). If it is not, configure [local upload mode](#ci-systems-that-are-not-supported-local-upload-mode) and tell the user what that means.
3. Do not set the CI, job and job-run identity options that `--help` lists (the `--ci-*`, `--job-*` and `--job-run-*` flags). On a supported CI `allurectl` fills them in by itself. On an unsupported CI the answer is local upload mode, not hand-made identifiers. There are exactly three exceptions: `ALLURE_JOB_UID` on [GitHub Enterprise Server](#github-enterprise-server), `ALLURE_JOB_UID` on [TeamCity with allurectl](#teamcity-with-allurectl), and the set of variables in the Task manifest for [Tekton](#tekton). In each case the values are set exactly as shown there and nothing else is added.
4. In CI wrap the test command with `allurectl watch`. Use `allurectl upload` only when the result files are already completely written.
5. The API token is a secret. Read it from the CI secret store or from an environment variable that the user sets. Never write it into a pipeline file, a script, a commit, or your reply.
6. Global flags (`--endpoint`, `--token`, `--log`, `--insecure`, `--output`) go before the subcommand: `allurectl --log debug watch -- <test command>`.

## Install

Download the binary for the operating system and CPU architecture from the latest release and make it executable.

| OS, architecture | File |
| --- | --- |
| macOS Intel | `allurectl_darwin_amd64` |
| macOS Apple silicon | `allurectl_darwin_arm64` |
| Linux, 32-bit CPU | `allurectl_linux_386` |
| Linux, 64-bit CPU | `allurectl_linux_amd64` |
| Linux, 64-bit ARM CPU | `allurectl_linux_arm64` |
| IBM POWER 64-bit, big-endian | `allurectl_linux_ppc64` |
| IBM POWER 64-bit, little-endian | `allurectl_linux_ppc64le` |
| Windows, 32-bit | `allurectl_windows_386.exe` |
| Windows, 64-bit | `allurectl_windows_amd64.exe` |
| Windows, 64-bit ARM CPU | `allurectl_windows_arm64.exe` |

The download URL is `https://github.com/allure-framework/allurectl/releases/latest/download/<file>`. Pick the file by the operating system and CPU of the machine or container where the tests run, not of the machine you are working on (check with `uname -sm` on Linux and macOS). The pipeline examples below use `allurectl_linux_amd64`: replace it when the runner is different.

```bash
curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl
chmod +x allurectl
./allurectl --version
```

If the image has no `curl`, use `wget ... -O ./allurectl`, or bake the binary into the runner image. On GitHub Actions use the `allure-framework/setup-allurectl@v1` action instead of downloading.

## Configuration

These values are the same for `watch` and `upload`. Prefer environment variables: they are easier to audit and reuse than flags.

| Environment variable | Flag | Meaning |
| --- | --- | --- |
| `ALLURE_ENDPOINT` | `-e`, `--endpoint` | Base URL of the Allure TestOps instance |
| `ALLURE_TOKEN` | `-t`, `--token` | Allure TestOps API token |
| `ALLURE_PROJECT_ID` | `--project-id` | Numeric Allure TestOps project ID |
| `ALLURE_RESULTS` | `--results` (for `watch`), positional `<directory>` (for `upload`) | Directory where the test framework writes Allure result files. Several directories: separate with commas or use a wildcard pattern |
| `ALLURE_LAUNCH_NAME` | `--launch-name` | Launch name, optional. A random `Launch-XXXXXXXX` name is used when it is not set |
| `ALLURE_LAUNCH_TAGS` | `--launch-tags` | Comma-separated launch tags, optional |

What you need from the user, ask if it is missing:

- the Allure TestOps URL and the numeric project ID;
- an API token. The user creates it in Allure TestOps: avatar menu, API Tokens, + Token. The user stores it as a CI secret named `ALLURE_TOKEN`, or exports it in their own shell for a local run;
- the path where the test framework writes Allure results (for example `build/allure-results`, `target/allure-results`, `allure-results`). The Allure adapter of the test framework must already be configured, `allurectl` only uploads what the adapter writes.

## Choosing the mode

`allurectl` works in two modes.

- CI-aware mode. `allurectl` runs inside a natively supported CI system and takes the pipeline context from the environment. Allure TestOps links the launch to the CI job and to the pipeline run, and can start and rerun the pipeline.
- Local upload mode. `allurectl` runs outside a supported CI context: on a workstation, or in a CI system that is not supported. The result is a standalone launch. There is no link to a pipeline, and Allure TestOps cannot start or rerun anything.

| Situation | What to do |
| --- | --- |
| Tests are about to run and you control the test command | `allurectl watch -- <test command>` |
| Tests have already finished, result files are on disk | `allurectl upload <directory>` |
| Results directory already has files before `watch` starts | Clean the directory first, or use `upload` after the run. `watch` is for files that appear while it is running |
| CI system from the supported list | CI-aware mode, see the pipeline for that CI below |
| Any other CI system | Local upload mode, see below |

## Upload from a local machine

Check the connection first. `auth login` only validates the endpoint and the token, it uploads nothing.

```bash
export ALLURE_ENDPOINT=https://testops.example.com
export ALLURE_TOKEN=<token from the user's environment>
./allurectl auth login
```

Run the tests and stream results while they run:

```bash
export ALLURE_ENDPOINT=https://testops.example.com
export ALLURE_PROJECT_ID=100
export ALLURE_RESULTS=build/allure-results
export ALLURE_LAUNCH_NAME="Local run"

./allurectl watch -- ./gradlew clean test
```

Upload results that already exist:

```bash
export ALLURE_ENDPOINT=https://testops.example.com
export ALLURE_PROJECT_ID=100
export ALLURE_LAUNCH_NAME="Local verification run"

./allurectl upload path/to/allure-results
```

The same as a one-off command with flags:

```bash
./allurectl upload \
  --endpoint https://testops.example.com \
  --project-id 100 \
  --launch-name "Local verification run" \
  path/to/allure-results
```

In these examples `ALLURE_TOKEN` is expected to be already exported by the user. Do not ask the user to paste the token into the chat and do not put it into a command line that ends up in shell history or in a file.

At the end of the log `allurectl` prints a link to the launch (`Report link: ...`). Give this link to the user.

## Natively supported CI systems

This is the complete list. Each line links to the public setup page.

| CI system | How results are uploaded | Setup page |
| --- | --- | --- |
| GitHub Actions (github.com and GitHub Enterprise Server) | `allurectl` | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/github/ |
| GitLab CI | `allurectl` | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/gitlab/ |
| Azure DevOps (Azure Pipelines) | `allurectl` | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/azure-devops-ci/ |
| Bitbucket Pipelines | `allurectl` | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/bitbucket-ci/ |
| CircleCI | `allurectl` | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/circleci/ |
| AWS CodePipeline / CodeBuild | `allurectl` | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/aws-codepipeline/ |
| Tekton | `allurectl`, with the pipeline context mapped through Task and Pipeline parameters | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/tekton/ |
| Jenkins | Allure TestOps Jenkins plugin (`withAllureUpload`), or `allurectl`. The plugin has to be installed in both cases if Allure TestOps must start pipelines | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/jenkins/ |
| TeamCity | Allure TestOps TeamCity plugin, or `allurectl` | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/teamcity/ |
| Bamboo | Allure TestOps Bamboo plugin, or `allurectl` | https://docs.qameta.io/allure-testops/reference/integrations/ci-systems/bamboo/ |

For Jenkins, TeamCity and Bamboo there are two ways to upload results: the Allure TestOps plugin for that CI, and `allurectl`. Do not pick one silently. Offer both to the user with the differences from [Jenkins, TeamCity, Bamboo: plugin or allurectl](#jenkins-teamcity-bamboo-plugin-or-allurectl) and configure the one the user chooses.

Any CI system that is not in this table is not natively supported. Go to [local upload mode](#ci-systems-that-are-not-supported-local-upload-mode).

## What every allurectl pipeline needs

1. `ALLURE_TOKEN` stored as a secret in the CI system by the user, and visible to the step that runs `allurectl`.
2. `ALLURE_ENDPOINT`, `ALLURE_PROJECT_ID` and `ALLURE_RESULTS` set as plain variables.
3. The `allurectl` binary available in the step.
4. The test command wrapped: `allurectl watch -- <test command>`. The wrapper must surround the command that really runs the tests. If the raw test command stays in the pipeline, nothing is uploaded.
5. A clean workspace for every run. On runners and agents that keep the workspace between runs, untracked files must be removed before the tests start. Otherwise a `testplan.json` left by a previous run started from Allure TestOps is used again, and old result files are uploaded into the new launch.
6. The Allure TestOps service variables (`ALLURE_JOB_RUN_ID`, and `ALLURE_USERNAME` where the CI requires it) passed through to `allurectl` untouched. Allure TestOps sends them when it starts the pipeline. When `ALLURE_JOB_RUN_ID` is present and not empty, `allurectl` uploads into the launch that Allure TestOps created. When it is empty, `allurectl` creates a new launch. Declare them exactly as in the example for the CI, with empty defaults, and never invent values for them.

Keep the pipeline's own handling of test failures. Do not add `|| true` or similar constructs around the wrapped command to make the step green.

## Pipelines

Replace `./gradlew clean test` and `build/allure-results` with the real test command and results path of the project.

### GitHub Actions

```yaml
name: tests

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      ALLURE_JOB_RUN_ID:
        description: "Allure TestOps service parameter. Leave blank."
        required: false
      ALLURE_USERNAME:
        description: "Allure TestOps service parameter. Leave blank."
        required: false

env:
  ALLURE_ENDPOINT: https://testops.example.com
  ALLURE_PROJECT_ID: 1
  ALLURE_RESULTS: build/allure-results
  ALLURE_JOB_RUN_ID: ${{ github.event.inputs.ALLURE_JOB_RUN_ID }}
  ALLURE_USERNAME: ${{ github.event.inputs.ALLURE_USERNAME }}

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install and configure allurectl
        uses: allure-framework/setup-allurectl@v1
        with:
          allure-endpoint: ${{ env.ALLURE_ENDPOINT }}
          allure-project-id: ${{ env.ALLURE_PROJECT_ID }}
          allure-token: ${{ secrets.ALLURE_TOKEN }}
      - name: Run tests
        run: allurectl watch -- ./gradlew clean test
```

Notes:

- The token is the repository secret `ALLURE_TOKEN` (Settings, Secrets and variables, Actions).
- One job in Allure TestOps corresponds to one GitHub workflow, one launch corresponds to one workflow run.
- `ALLURE_JOB_RUN_ID` and `ALLURE_USERNAME` must be declared as `required: false` inputs of `workflow_dispatch`. If they are missing, GitHub answers 422 when Allure TestOps starts the workflow. Every parameter configured on the Allure TestOps job must also exist as a `workflow_dispatch` input. Branch is the exception, it is not an input.
- After any change of `workflow_dispatch` inputs the user has to click "Update job from the build server" on the job in Allure TestOps.
- On GitHub Enterprise Server the Marketplace action is not available and this workflow is not enough. Use the workflow from the next section.

### GitHub Enterprise Server

First find out where the workflow runs. If the repository is on a GitHub Enterprise Server instance (the server URL is not `https://github.com`), the workflow differs from the github.com one in two ways, and both are required.

1. `allurectl` is downloaded directly, because the `allure-framework/setup-allurectl` Marketplace action cannot be used on GitHub Enterprise Server.
2. The workflow computes the job identifier itself and passes it to `allurectl` in the `ALLURE_JOB_UID` environment variable. This step is mandatory. Without it the workflow cannot be linked to the Allure TestOps job, and Allure TestOps cannot start or rerun the workflow.

```yaml
name: tests

on:
  workflow_dispatch:
    inputs:
      ALLURE_JOB_RUN_ID:
        description: "Allure TestOps service parameter. Leave blank."
        required: false
      ALLURE_USERNAME:
        description: "Allure TestOps service parameter. Leave blank."
        required: false

env:
  ALLURE_ENDPOINT: ${{ secrets.ALLURE_ENDPOINT }}
  ALLURE_TOKEN: ${{ secrets.ALLURE_TOKEN }}
  ALLURE_PROJECT_ID: ${{ secrets.ALLURE_PROJECT_ID }}
  ALLURE_RESULTS: build/allure-results
  ALLURE_JOB_RUN_ID: ${{ github.event.inputs.ALLURE_JOB_RUN_ID }}
  ALLURE_USERNAME: ${{ github.event.inputs.ALLURE_USERNAME }}

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/github-script@v4
        id: allure-job-uid
        with:
          result-encoding: string
          script: |
            const result = await github.actions.getWorkflowRun({
              owner: context.repo.owner,
              repo: context.repo.repo,
              run_id: context.runId,
            });
            return `${context.repo.owner}/${context.repo.repo}/actions/workflows/${result.data.workflow_id}`
      - name: Download allurectl
        run: |
          wget https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -O ./allurectl
          chmod +x ./allurectl
      - name: Run tests
        env:
          ALLURE_JOB_UID: ${{ steps.allure-job-uid.outputs.result }}
        run: ./allurectl watch -- ./gradlew clean test
```

How to do it step by step:

1. Add the `allure-job-uid` step before the step that runs the tests. It asks the GitHub API for the current workflow run and returns a string of the form `<owner>/<repo>/actions/workflows/<workflow_id>`, where `<workflow_id>` is the numeric ID of the workflow. Keep `id: allure-job-uid` and `result-encoding: string`, the next step reads the value by this id as a plain string.
2. In the step that runs `allurectl`, set `ALLURE_JOB_UID: ${{ steps.allure-job-uid.outputs.result }}` in `env`. The variable must be set in the `env` of the same step where `allurectl watch` runs. A value set in another step's `env` is not visible to `allurectl`, and the workflow-level and job-level `env` blocks cannot read step outputs.
3. Download `allurectl` in its own step and call it as `./allurectl`. If the runners have no access to github.com, the binary has to be preinstalled on the runner image or taken from an internal mirror: ask the user where it is available.
4. Keep the `workflow_dispatch` inputs `ALLURE_JOB_RUN_ID` and `ALLURE_USERNAME` and their mapping in `env`, exactly as for github.com.
5. Wrap the test command with `./allurectl watch --`, as everywhere else.

Do not change the format of the value and do not type it by hand: it must be computed by the step in every run. Do not add any other `ALLURE_JOB_*` or `ALLURE_CI_*` variables next to it.

Notes:

- `ALLURE_ENDPOINT`, `ALLURE_TOKEN` and `ALLURE_PROJECT_ID` are repository secrets in this example. The token must be a secret, the other two may also be plain values in `env`.
- The script is written for `actions/github-script@v4`. From version 5 of that action the same call is `github.rest.actions.getWorkflowRun(...)`. If the version is changed, change the call too, otherwise the step fails and `ALLURE_JOB_UID` stays empty.
- The step calls the GitHub API with the workflow token, so the token needs read access to Actions. If the step fails with a permission error, add `permissions: actions: read` (together with `contents: read` for checkout) to the job.
- If a launch appears in Allure TestOps but the job cannot be started or rerun from Allure TestOps, first check in the workflow log that the `allure-job-uid` step succeeded and that `ALLURE_JOB_UID` is set on the step that runs `allurectl`.
- The Allure TestOps side is the same as for github.com, except the integration endpoints: the global GitHub integration uses the GitHub Enterprise Server URL as the endpoint and `<GHES URL>/api/v3` as the endpoint for API calls. The user configures this in the Allure TestOps UI.

### GitLab CI

```yaml
test:
  stage: test
  image: gradle
  variables:
    ALLURE_ENDPOINT: https://testops.example.com
    ALLURE_PROJECT_ID: 1
    ALLURE_RESULTS: build/allure-results
  before_script:
    - curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl
    - chmod +x allurectl
  script:
    - ./allurectl watch -- gradle clean test
```

Notes:

- The token is a masked CI/CD variable `ALLURE_TOKEN` (project Settings, CI/CD, Variables).
- Extra pipeline parameters (browser, product version and so on) go to the `variables` block with defaults. Allure TestOps can override them when it starts the pipeline.
- Only CI/CD `variables` are supported for passing parameters. GitLab pipeline `inputs` (`spec: inputs` and `$[[ inputs.<name> ]]`) are not supported: Allure TestOps passes parameters to a GitLab pipeline as variables and cannot fill inputs. Declare every parameter that Allure TestOps has to set or read as a variable. If the existing pipeline takes a parameter as an input, do not map it to the Allure TestOps job, add a variable for it instead, and tell the user about this limitation.

### Azure DevOps

```yaml
steps:
  - bash: curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl && chmod +x allurectl
    displayName: Download allurectl
  - script: ./allurectl watch -- ./gradlew clean test
    displayName: Run tests
    env:
      ALLURE_ENDPOINT: https://testops.example.com
      ALLURE_PROJECT_ID: 1
      ALLURE_RESULTS: build/allure-results
      ALLURE_TOKEN: $(ALLURE_TOKEN)
```

Notes:

- The user creates two pipeline variables in the Azure DevOps UI (pipeline, Edit, Variables). `ALLURE_TOKEN`: secret, not overridable. `ALLURE_JOB_RUN_ID`: empty value, not secret, "Let users override this value when running this pipeline" checked.
- A secret variable is not exposed to scripts automatically, so the `ALLURE_TOKEN: $(ALLURE_TOKEN)` mapping in `env` is required.

### Bitbucket Pipelines

```yaml
image: gradle:8.10-jdk21

pipelines:
  custom:
    test:
      - variables:
          - name: ALLURE_LAUNCH_NAME
            default: ""
          - name: ALLURE_LAUNCH_TAGS
            default: "bitbucket, junit5"
      - step:
          name: Run tests
          script:
            - wget https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -O ./allurectl
            - chmod +x ./allurectl
            - ALLURE_LAUNCH_NAME="${BITBUCKET_REPO_FULL_NAME} - $BITBUCKET_BUILD_NUMBER"
            - ./allurectl watch -- gradle --no-daemon clean test
```

Notes:

- `ALLURE_ENDPOINT` is a workspace or repository variable. `ALLURE_TOKEN` (secured), `ALLURE_PROJECT_ID` and `ALLURE_RESULTS` are repository variables.
- A named pipeline under `custom:` (`test` in the example) is required if Allure TestOps should start the pipeline. Its `variables` block must contain at least `ALLURE_LAUNCH_NAME` and `ALLURE_LAUNCH_TAGS`.
- In the Allure TestOps job the user sets the default of the `CustomName` parameter to the name of that custom pipeline.

### CircleCI

```yaml
version: 2.1

parameters:
  ALLURE_JOB_RUN_ID:
    type: string
    default: ""
  ALLURE_USERNAME:
    type: string
    default: ""

jobs:
  test:
    docker:
      - image: cimg/openjdk:17.0
    environment:
      ALLURE_ENDPOINT: https://testops.example.com
      ALLURE_PROJECT_ID: "1"
      ALLURE_RESULTS: build/allure-results
      ALLURE_JOB_RUN_ID: << pipeline.parameters.ALLURE_JOB_RUN_ID >>
      ALLURE_USERNAME: << pipeline.parameters.ALLURE_USERNAME >>
    steps:
      - checkout
      - run:
          name: Download allurectl
          command: curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl && chmod +x allurectl
      - run:
          name: Run tests
          command: ./allurectl watch -- ./gradlew clean test

workflows:
  test:
    jobs:
      - test
```

Notes:

- The token is a project environment variable `ALLURE_TOKEN` in the CircleCI project settings.
- Keep `version: 2.1`. `ALLURE_JOB_RUN_ID` and `ALLURE_USERNAME` must always be declared as optional string parameters and referenced in the job `environment`.

### AWS CodePipeline / CodeBuild

```yaml
version: 0.2

phases:
  install:
    commands:
      - curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl
      - chmod +x allurectl
  build:
    commands:
      - ./allurectl watch -- ./gradlew clean test
```

Notes:

- `ALLURE_ENDPOINT`, `ALLURE_TOKEN`, `ALLURE_PROJECT_ID`, `ALLURE_RESULTS` and optionally `ALLURE_LAUNCH_NAME` are environment variables of the CodeBuild project that runs the tests. If the token is kept in AWS Secrets Manager, it must be injected into the same environment where the test command runs.
- The CodeBuild service role needs `codebuild:BatchGetBuilds`, `codebuild:BatchGetBuildBatches` and `codepipeline:GetPipelineState`, scoped to the relevant resources. This is an IAM change: describe it to the user, do not apply it on your own.

### Tekton

One job in Allure TestOps corresponds to one Tekton Pipeline, one launch corresponds to one PipelineRun.

Tekton does not put the pipeline context into environment variables. The Pipeline passes it to the Task as parameters, and the Task turns the parameters into the `ALLURE_*` environment variables of the step that runs `allurectl`. Three manifests are needed: a Task, a Pipeline and a PipelineRun. The examples are intentionally simplified: adapt the image, workspace, security context and parameter strategy to the policies of the cluster, and keep the parameter and variable mapping as it is.

#### Task

File `allure-example-test-task.yaml`. The Task downloads `allurectl`, runs the tests with `allurectl watch` and receives the environment variables.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: allure-example-test
spec:
  params:
    - name: ALLURE_JOB_RUN_ID
      type: string
    - name: namespace
      type: string
    - name: pipelineName
      type: string
    - name: pipelineRunName
      type: string
  steps:
    - name: test
      image: gradle:jdk8
      script: |
        git clone https://github.com/reponame/allure-example.git && cd allure-example
        wget -q -O ./allurectl https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 && chmod +x ./allurectl
        ./allurectl watch -- ./gradlew clean test
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        seccompProfile:
          type: RuntimeDefault
      env:
        - name: ALLURE_ENDPOINT
          value: "https://testops.example.com"
        - name: ALLURE_TOKEN
          valueFrom:
            secretKeyRef:
              name: allure-testops
              key: token
        - name: ALLURE_PROJECT_ID
          value: "1"
        - name: ALLURE_RESULTS
          value: "build/allure-results"
        - name: ALLURE_CI_TYPE
          value: "tekton"
        - name: ALLURE_LAUNCH_NAME
          value: "$(params.pipelineName) - $(params.pipelineRunName)"
        - name: ALLURE_JOB_URL
          value: "https://tekton.example.com/#/namespaces/$(params.namespace)/pipelines/$(params.pipelineName)"
        - name: ALLURE_JOB_UID
          value: "$(params.namespace)/$(params.pipelineName)"
        - name: ALLURE_JOB_NAME
          value: "$(params.pipelineName)"
        - name: ALLURE_JOB_RUN_ID
          value: "$(params.ALLURE_JOB_RUN_ID)"
        - name: ALLURE_JOB_RUN_URL
          value: "https://tekton.example.com/#/namespaces/$(params.namespace)/pipelineruns/$(params.pipelineRunName)"
        - name: ALLURE_JOB_RUN_UID
          value: "$(params.pipelineRunName)"
        - name: ALLURE_JOB_RUN_NAME
          value: "$(params.pipelineRunName)"
        - name: ALLURE_TESTPLAN_PATH
          value: "./testplan.json"
```

The only difference from the manifest in the public documentation is `ALLURE_TOKEN`: here it is read from a Kubernetes Secret (`allure-testops`, key `token`) that the user creates in the namespace of the pipeline. Do not put the token into a manifest or a parameter value.

Environment variables of the step:

| Variable | Required | Description |
| --- | --- | --- |
| `ALLURE_ENDPOINT` | Yes | Allure TestOps instance URL with protocol |
| `ALLURE_TOKEN` | Yes | Allure TestOps API token |
| `ALLURE_PROJECT_ID` | Yes | Numeric Allure TestOps project ID |
| `ALLURE_RESULTS` | Yes | Path to the Allure results directory |
| `ALLURE_CI_TYPE` | Yes | Always `tekton` |
| `ALLURE_JOB_RUN_ID` | Yes | Passed from the Pipeline, links the run to the correct Allure TestOps launch |
| `ALLURE_LAUNCH_NAME` | No | Launch name template |
| `ALLURE_JOB_URL` | No | URL of the Pipeline in the Tekton dashboard |
| `ALLURE_JOB_UID` | No | Stable unique identifier of the Pipeline, `namespace/pipelineName` |
| `ALLURE_JOB_NAME` | No | Pipeline name |
| `ALLURE_JOB_RUN_URL` | No | URL of the PipelineRun in the Tekton dashboard |
| `ALLURE_JOB_RUN_UID` | No | Stable unique identifier of the PipelineRun |
| `ALLURE_JOB_RUN_NAME` | No | PipelineRun name |
| `ALLURE_TESTPLAN_PATH` | No | Path to the test plan file, required for selective execution |

The variables marked "No" are optional for the upload itself, but they give Allure TestOps the links and the job identity. Keep all of them, with the values from the manifest. They are built only from the `$(params...)` substitutions: do not replace them with fixed strings and do not invent other values.

Apply the Task:

```bash
kubectl apply -f allure-example-test-task.yaml
```

#### Pipeline

File `allure-example-pipeline.yaml`. The Pipeline declares `ALLURE_JOB_RUN_ID` as a parameter with an empty default and passes it down to the Task together with the pipeline context.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: allure-example
spec:
  params:
    - description: Allure Job Run ID
      name: ALLURE_JOB_RUN_ID
      type: string
      default: ""
  tasks:
    - name: test
      taskRef:
        name: allure-example-test
      params:
        - name: ALLURE_JOB_RUN_ID
          value: "$(params.ALLURE_JOB_RUN_ID)"
        - name: pipelineName
          value: "$(context.pipeline.name)"
        - name: pipelineRunName
          value: "$(context.pipelineRun.name)"
        - name: namespace
          value: "$(context.pipelineRun.namespace)"
```

```bash
kubectl apply -f allure-example-pipeline.yaml
```

#### PipelineRun

File `allure-example-pipelinerun.yaml`. For a run started by hand `ALLURE_JOB_RUN_ID` stays empty. Allure TestOps fills it when it starts the pipeline.

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: allure-example-run-
  namespace: tekton-pipelines
spec:
  pipelineRef:
    name: allure-example
  params:
    - name: ALLURE_JOB_RUN_ID
      value: ""
  serviceAccountName: default
  podTemplate:
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      fsGroup: 1000
      seccompProfile:
        type: RuntimeDefault
```

Submit the PipelineRun and look at the runs:

```bash
kubectl create -f allure-example-pipelinerun.yaml
```

```bash
kubectl get pipelineruns -n tekton-pipelines
```

#### What to change and what to keep

Change:

- the Task, Pipeline and PipelineRun names and the namespace;
- the step image, the repository URL and the test command after `./allurectl watch --`;
- `ALLURE_ENDPOINT`, `ALLURE_PROJECT_ID`, `ALLURE_RESULTS`;
- `https://tekton.example.com` in `ALLURE_JOB_URL` and `ALLURE_JOB_RUN_URL`: the base URL of the Tekton dashboard that users open in a browser;
- the name and key of the Secret with the token.

Keep:

- the `ALLURE_JOB_RUN_ID` chain: PipelineRun parameter, Pipeline parameter with the empty default, Task parameter, step environment variable. If one link is missing, a run started from Allure TestOps uploads into a new launch;
- the `pipelineName`, `pipelineRunName` and `namespace` parameters filled from `$(context.pipeline.name)`, `$(context.pipelineRun.name)` and `$(context.pipelineRun.namespace)`;
- `ALLURE_CI_TYPE` equal to `tekton`;
- the Pipeline name after the first upload. Tekton has no automatic pipeline discovery, so renaming the Pipeline breaks the job identity in Allure TestOps.

Do not delete existing PipelineRuns to tidy up unless the user asks for it.

#### Check and the trigger path

After the PipelineRun completes, the launch with results must exist in Allure TestOps, and a test result must link back to the PipelineRun URL.

Starting pipelines from Allure TestOps needs the Tekton dashboard deployed and reachable from the Allure TestOps instance, and at least one successful upload, because the job is created from the first upload. The rest is done by the user in the Allure TestOps UI: the global Tekton integration with the dashboard base URL as the endpoint, the integration in the project, and the job settings (build server, "Job can be used to run tests"). Additional runtime parameters are declared in the Pipeline, passed to the Task as environment variables and mapped in the project environment settings.

| Symptom | Check |
| --- | --- |
| Results arrive but the Tekton run is not linked | `ALLURE_JOB_RUN_ID` or the job variables are missing in the step environment. Check the pass-through PipelineRun, Pipeline, Task |
| Allure TestOps cannot start the pipeline | The Tekton dashboard is reachable from the Allure TestOps instance, the project integration uses the right authentication mode, the job points at the right Tekton integration and Pipeline |

### Jenkins, TeamCity, Bamboo: plugin or allurectl

These three CI systems have an Allure TestOps plugin, and `allurectl` works there as well. When the task is to set up the upload on one of them, give the user both options and the differences below, then ask which one to use. If the user has already named an option, use it and still mention the limitation that applies to it.

| CI | Plugin | `allurectl` |
| --- | --- | --- |
| Jenkins | Upload with the `withAllureUpload(...) { ... }` wrapper. Allure TestOps can start and rerun builds | Upload with `allurectl watch`. Allure TestOps can start and rerun builds only if the plugin is also installed in Jenkins |
| TeamCity | Upload with the plugin build feature. Allure TestOps can start and rerun builds | Upload with `allurectl watch` inside the build. Allure TestOps can start and rerun builds, the plugin is not needed for that. One parameter, `env.ALLURE_JOB_UID`, has to be set explicitly |
| Bamboo | Upload with the plugin task settings. Allure TestOps can start and rerun builds | Upload with `allurectl watch` in a script task. Starting builds from Allure TestOps is documented only for the plugin |

How to help the user choose:

- The plugin needs permission to install plugins and to change global settings of the CI server, and it has requirements for the CI server and Java versions (see the setup page). `allurectl` needs only a pipeline change and a secret.
- Starting and rerunning tests from Allure TestOps depends on the CI system. On Jenkins the plugin has to be installed on the CI server for that, even when the upload itself is done by `allurectl`. On TeamCity it works with `allurectl` inside the build and without the plugin. For Bamboo it is documented only for the plugin.
- Use one upload mechanism for a given test step. Do not wrap the same tests with both the plugin and `allurectl watch`.

For the plugin option follow the setup page of the CI system. For Jenkins the test step is wrapped with `withAllureUpload(...) { ... }`, and the wrapper must surround the step that really runs the tests.

### Jenkins with allurectl

Important: in the current implementation Allure TestOps starts Jenkins pipelines through the Allure TestOps Jenkins plugin. If `allurectl` is used for the upload and the plugin is not installed in Jenkins, results are uploaded, but pipelines cannot be triggered from Allure TestOps. Tell this to the user before configuring this option. If the user needs to run or rerun tests from Allure TestOps, the plugin has to be installed in Jenkins (section "Install the Jenkins plugin" of the setup page) and the trigger path has to be configured as described there, even though the pipeline uploads with `allurectl`.

```groovy
pipeline {
    agent any
    environment {
        ALLURE_ENDPOINT = 'https://testops.example.com'
        ALLURE_PROJECT_ID = '1'
        ALLURE_RESULTS = 'build/allure-results'
        ALLURE_TOKEN = credentials('allure-testops-token')
    }
    stages {
        stage('Test') {
            steps {
                sh 'curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl && chmod +x allurectl'
                sh './allurectl watch -- ./gradlew clean test'
            }
        }
    }
}
```

Notes:

- The token is a Jenkins credential of the "Secret text" kind that the user creates. `credentials('<credential id>')` puts it into `ALLURE_TOKEN` and masks it in the log. Do not put the token into the Jenkinsfile.
- Do not set job or build identifiers. `allurectl` takes the job and build context from the Jenkins build environment.
- The `sh` step that runs the tests must be the one wrapped with `./allurectl watch --`.
- If the tests run inside a container started by the pipeline itself, see [Tests run in another container or on another host](#tests-run-in-another-container-or-on-another-host).
- After the trigger path is configured, start one run from Allure TestOps and check that the results arrive in the launch created by Allure TestOps and not in a new one. If a new launch is created, send the user to support: https://help.qameta.io.

### TeamCity with allurectl

When `allurectl` is used inside the build, both directions work: results are uploaded to Allure TestOps, and Allure TestOps can start and rerun TeamCity builds. The TeamCity plugin is not required for this option.

One job in Allure TestOps corresponds to one TeamCity build configuration, one launch corresponds to one TeamCity build.

`allurectl` recognizes TeamCity by itself and collects the build context (job name, build number, build URL) from the build environment. Use the latest release, automatic detection of TeamCity needs `allurectl` 2.20.0 or newer. There is one value it does not get right on TeamCity at the moment: the job UID. It has to be passed explicitly, in the parameters of the build configuration:

```
env.ALLURE_JOB_UID = %system.teamcity.buildType.id%
```

This parameter is mandatory. Without it the job in Allure TestOps gets a wrong identifier. Do not set any other `ALLURE_JOB_*` or `ALLURE_CI_*` parameter, the rest is detected automatically.

#### How environment variables reach allurectl in TeamCity

`allurectl` reads its settings from the environment of the process that starts it. TeamCity has three kinds of parameters, and only one of them ends up in that environment:

| Kind | Name looks like | Visible to `allurectl` |
| --- | --- | --- |
| Environment variable | `env.ALLURE_ENDPOINT` | Yes, as `ALLURE_ENDPOINT` (the `env.` prefix is removed) |
| System property | `system.teamcity.buildType.id` | No |
| Configuration parameter | `teamcity.build.id` | No |

The build configuration ID lives in the system property `system.teamcity.buildType.id`, which is not in the environment of the build step. To hand it to `allurectl` you create an `env.` parameter whose value is a reference to that property in percent signs. TeamCity resolves the reference when the build starts, and the build step receives a normal environment variable `ALLURE_JOB_UID` with the real build configuration ID.

#### Parameters to create

Step 1. The user creates an Allure TestOps API token (avatar menu, API Tokens, + Token).

Step 2. Add the following parameters to the build configuration that runs the tests. In the TeamCity UI: build configuration settings, Parameters, Add new parameter. Enter the name with the `env.` prefix exactly as in the table.

| Name | Value | Notes |
| --- | --- | --- |
| `env.ALLURE_ENDPOINT` | URL of the Allure TestOps instance | Constant |
| `env.ALLURE_PROJECT_ID` | Numeric Allure TestOps project ID | Constant |
| `env.ALLURE_TOKEN` | Allure TestOps API token | Secret, parameter of the Password type, entered by the user |
| `env.ALLURE_RESULTS` | Path to the Allure results directory | Constant |
| `env.ALLURE_JOB_UID` | `%system.teamcity.buildType.id%` | Mandatory. Reference, entered literally with the percent signs |
| `env.ALLURE_LAUNCH_NAME` | Launch name, for example `%system.teamcity.buildConfName% - #%build.number%` | Optional |
| `env.ALLURE_LAUNCH_TAGS` | Comma-separated tags | Optional |

Rules for these parameters:

- Enter the value of `env.ALLURE_JOB_UID` exactly as `%system.teamcity.buildType.id%`. Never replace it with a fixed string, the configuration name, a random value or anything computed in the build script. TeamCity substitutes the ID of the build configuration, which stays the same for every build of that configuration. This is the identifier Allure TestOps uses to attach uploads to the same job, so do not change it after the first upload: a different value creates a different job in Allure TestOps.
- A reference to a parameter that does not exist blocks the build: TeamCity treats it as a requirement that no agent satisfies, and the build stays in the queue. Copy the name from the table and check for typos.
- `ALLURE_TOKEN` is a secret. Create it as a parameter of the Password type so that TeamCity hides it in the UI and in the build log. The user enters the value, you do not.
- The parameters may also be defined once at project level. Project parameters are inherited by every build configuration, and the reference in `env.ALLURE_JOB_UID` is still resolved separately for each build configuration, so each one gets its own job UID.

If the TeamCity settings are stored in the repository (versioned settings, Kotlin DSL), the same parameters go into the `params` block of the build configuration:

```kotlin
params {
    param("env.ALLURE_ENDPOINT", "https://testops.example.com")
    param("env.ALLURE_PROJECT_ID", "1")
    param("env.ALLURE_RESULTS", "build/allure-results")
    param("env.ALLURE_JOB_UID", "%system.teamcity.buildType.id%")
    password("env.ALLURE_TOKEN", "credentialsJSON:<id of the secure value generated in TeamCity>")
}
```

The secure value for `password(...)` is generated by the user in the TeamCity UI. Never write the token itself into the settings file.

#### Clean checkout

Step 3. Enable clean checkout for the build configuration that runs the tests. In the TeamCity UI: build configuration settings, Version Control Settings, show advanced options, check "Clean build: delete all files in the checkout directory before the build".

In Kotlin DSL:

```kotlin
vcs {
    root(DslContext.settingsRoot)
    cleanCheckout = true
}
```

This is required. Every build must start from a fresh copy of the repository with no untracked files left from earlier builds. The reason:

- When a run is started from Allure TestOps with a selection of tests, a `testplan.json` file appears in the working directory of the build. TeamCity reuses the checkout directory between builds, so the file stays there.
- If the next run is started without a selection (for example a full run from Allure TestOps), the old `testplan.json` is still in place and is used again. The build silently runs the tests selected for the previous run instead of the tests it was asked to run.
- The same happens with result files of the previous build: they are uploaded into the launch of the new build.

Clean checkout removes the whole checkout directory before the build and fetches the sources from Git again, so neither `testplan.json` nor old results survive. Deleting only the results directory in the script or relying on the `clean` task of the build tool is not enough, because `testplan.json` is not in the results directory.

If the user objects to a full re-checkout for every build (a very large repository), the acceptable alternative is the clean policy of the Git VCS root: in the VCS root settings set "Clean policy" to "Always" and "Clean files policy" to "All untracked files". It must be the option that removes all untracked files including ignored ones, because `testplan.json` and the results directory are usually listed in `.gitignore`. Do not choose anything weaker.

#### Build step

Step 4. Update the build step that runs the tests (build configuration settings, Build Steps, edit the step). Download `allurectl` and wrap the test command with `watch`:

```bash
curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl
chmod +x allurectl
./allurectl watch -- ./gradlew clean test
```

The same step in Kotlin DSL:

```kotlin
steps {
    script {
        name = "Run tests"
        scriptContent = """
            curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl
            chmod +x allurectl
            ./allurectl watch -- ./gradlew clean test
        """.trimIndent()
    }
}
```

Things that break the environment on the way to `allurectl`:

- The tests must be started by `allurectl` in a Command Line (script) step. If the tests stay in a separate Gradle, Maven or other runner step, `allurectl watch` does not wrap them. Move the test command into the script step after `--`.
- TeamCity substitutes `%name%` inside the script text too. A literal percent sign in the script has to be written as `%%`, otherwise TeamCity looks for a parameter with that name and the build cannot start.
- If the step uses the TeamCity option to run the step inside a Docker container, the `env.` parameters and the TeamCity build environment are passed into that container, and `allurectl` has to be downloaded and started inside it, as in the script above.
- If the script itself starts a container (`docker run`, `docker compose`) and the tests run there, nothing is passed automatically, neither the `ALLURE_*` variables nor the TeamCity build environment that `allurectl` uses for detection. Use the approach from [Tests run in another container or on another host](#tests-run-in-another-container-or-on-another-host).
- If build agents use a different operating system or architecture, download the matching binary, see [Install](#install). On a Windows agent the step is a Windows script and the binary is `allurectl_windows_amd64.exe`.

#### Check the result

Step 5. Run a build and check:

- Clean checkout is enabled and works: start a run from Allure TestOps with a few selected tests, then a full run. The second build must execute all tests, not the selection of the first one, and its launch must contain only its own results.
- On the page of the finished build, the Parameters tab shows the actual values of the environment variables. `ALLURE_JOB_UID` must contain the build configuration ID (for example `MyProject_Tests`) and no percent signs.
- The `allurectl` output at the start of the run says that the CI system was recognized, and the end of the build log contains the link to the launch.
- In Allure TestOps the launch is attached to a job, and repeated builds of the same build configuration go to the same job.
- `allurectl` reports that no CI system was detected, or a standalone launch is created: the tests run in a place that did not receive the build environment (see the container cases above), or an old `allurectl` version is used.
- A new job appears in Allure TestOps for every build, or the job has an unexpected identifier: `env.ALLURE_JOB_UID` is missing, misspelled or overridden. Recheck the parameter.
- After the trigger path is configured (see below), start one run from Allure TestOps and check that the build starts in TeamCity and that its results arrive in the launch created by Allure TestOps and not in a new one.

#### Starting and rerunning builds from Allure TestOps

Runs and reruns started from Allure TestOps work with this option, provided that the tests are started by `allurectl watch` inside the build and `env.ALLURE_JOB_UID` is set as described above. The Allure TestOps side is configured by the user in the UI, as described in the sections of the TeamCity setup page about triggering TeamCity builds and about passing parameters and branch values: the global TeamCity integration, the TeamCity access token in the project, and the job settings (build server, "Job can be used to run tests", parameters). The job itself appears in Allure TestOps after the first upload from TeamCity. List these steps for the user, you cannot do them from the pipeline.

If the job cannot be started from Allure TestOps, check first that the job UID is the build configuration ID (see the checks above), because Allure TestOps uses it to find the build configuration to start. If a started build uploads into a new launch, check that the tests run under `allurectl watch` in the build step and not in a container that did not receive the build environment.

### Bamboo with allurectl

The public setup page describes Bamboo only with the plugin, including starting builds from Allure TestOps. With `allurectl` the upload is configured like in any other pipeline:

1. The user stores the Allure TestOps token as a plan variable that Bamboo masks.
2. In the Script task that runs the tests, fill the "Environment variables" field of the task so that the `ALLURE_*` variables exist under their exact names: `ALLURE_ENDPOINT`, `ALLURE_PROJECT_ID`, `ALLURE_RESULTS`, and `ALLURE_TOKEN` mapped from the plan variable in the `NAME=${bamboo.<variable>}` form. Bamboo plan variables do not appear in the script under their own names, so this mapping is required.
3. In the script body download `allurectl` and wrap the test command: `./allurectl watch -- <test command>`.

Do not set job or build identifiers. If the user needs to start or rerun builds from Allure TestOps, use the plugin as described on the setup page.

## CI systems that are not supported: local upload mode

If the CI system is not in the supported list, `allurectl` works there exactly as on a workstation. Configure it like this and state the consequences to the user explicitly.

```bash
# ALLURE_TOKEN comes from the secret store of the CI system
export ALLURE_ENDPOINT=https://testops.example.com
export ALLURE_PROJECT_ID=100
export ALLURE_RESULTS=build/allure-results
export ALLURE_LAUNCH_NAME="<pipeline name> - <build number>"
export ALLURE_LAUNCH_TAGS="<branch>,<suite>"

curl -fsSL https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -o allurectl
chmod +x allurectl
./allurectl watch -- <test command>
```

What the user gets and does not get:

- Every run creates a standalone launch with the given name and tags.
- The launch is not linked to a CI job or to a pipeline run.
- Allure TestOps cannot start or rerun the pipeline, and selective runs started from Allure TestOps are not available.
- Build the launch name and tags from the build variables of that CI system (pipeline name, build number, branch), so launches can be found and filtered.

Do not try to turn this into CI-aware mode by passing CI, job or job-run identifiers by hand. If the user needs a full integration with a CI system that is not in the list, send them to support: https://help.qameta.io.

## Several launches from one CI build

In CI-aware mode one CI build (one workflow run, one pipeline run) corresponds to one launch. Everything uploaded during that build goes into the same launch. When Allure TestOps starts a job with several environments, the resulting pipeline runs also upload to one launch.

If the user asks for several separate launches from one CI build (for example a launch per stage, per test suite, per browser or per matrix cell of the same build):

1. Do not implement it.
2. Do not look for a workaround. This includes different launch names per step, `--launch-id`, `allurectl launch create`, `allurectl launch merge`, changing CI, job or job-run options, and hiding the CI environment from `allurectl`.
3. Tell the user that this scenario has to be discussed with Qameta support and give them the link: https://help.qameta.io.

## Tests run in another container or on another host

If the tests run outside the parent CI job (a Docker container, a remote VM, a nested runner, a downstream pipeline), the pipeline context has to be carried over. In the parent job:

```bash
ALLURE_CI_ENV=$(allurectl ci env --encode)
```

Pass `ALLURE_CI_ENV` together with the other `ALLURE_*` variables into the environment where the tests really run, make sure the results directory is visible there, and start `allurectl watch` there as usual. Without this the upload becomes an unrelated standalone launch.

## After the upload

Later pipeline steps can read the launch and job-run values, for example for a notification or a build summary:

```bash
export $(allurectl job-run env)
```

This provides `ALLURE_LAUNCH_URL` and `ALLURE_JOB_RUN_URL` among others.

To add issue links to the launch of the current CI run (`<integration_id>` is shown in the integration settings in Allure TestOps):

```bash
allurectl launch add-issue --current-launch --integration-id <integration_id> --issue-keys AE-1,AE-2
```

Allure TestOps creates and updates automated test cases from the uploaded results when the launch is closed. A launch is closed in the Allure TestOps UI or with `allurectl launch close <launch-id>`. Ask the user before closing or deleting anything. `allurectl launch delete` is irreversible, never run it on your own.

## Useful options

All of these are listed by `allurectl watch --help` or `allurectl upload --help`.

| Option | Command | Use |
| --- | --- | --- |
| `--timeout` | `watch`, `upload` | Timeout for execution. Default for `watch` is 1 hour, for `upload` 10 hours. Raise it for `watch` when the test suite can run longer than an hour |
| `--continue-on-error` | `watch` | Continue the watched command and keep its exit code when Allure TestOps service errors prevent uploading. Older releases and the public docs call this `--silent`, check `--help` of the installed version |
| `--fail-fast` | `watch` | Stop the watched command when Allure TestOps service errors make uploading unavailable. Use either this or `--continue-on-error`, not both |
| `--reopen-closed-launch` | `watch` | Reopen a closed launch before uploading into it |
| `--skip-too-big` | `watch`, `upload` | Skip files that Allure TestOps would not process (`*-result.json` larger than 2,000,000 bytes) |
| `--ignore-passed-test-attachments` | `watch`, `upload` | Do not upload attachments of passed tests |
| `--exclude-files` | `watch`, `upload` | Exclude files by name, Go-flavoured regular expression |
| `--stdout-max-buff-size` | `watch` | Raise when the run fails with `bufio.Scanner: token too long`. Default 1048576, increase gradually |
| `--results` | `watch` | Result directories, same as `ALLURE_RESULTS` |
| `--pb` | `upload` | Progress bar |
| `-i`, `--insecure` | global | Skip certificate validation. Only when the user confirms that the Allure TestOps instance uses a self-signed certificate |
| `--log` | global | Log level: `FATAL`, `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE` |

Leave the batching, concurrency and rate-limit options at their defaults unless support asks to change them.

## Verify and troubleshoot

After a run, read the end of the `allurectl` log.

- `Report link:` is the launch URL. Report it to the user.
- `ErrorsCount` must be 0. If it is not, rerun with `--log debug` placed before the subcommand.
- `Completed` is zero and errors are present: the results directory was populated before `watch` started. Use `upload`, or clean the directory before the run.
- Non-zero `Orphan Files` or `unknown`: attachments without a matching result file, or files that are not Allure results. Often the results path is wrong.

| Symptom | Check |
| --- | --- |
| No results in Allure TestOps | `ALLURE_ENDPOINT`, `ALLURE_TOKEN`, `ALLURE_PROJECT_ID` point to the right project. `ALLURE_RESULTS` is the directory the framework really writes to. The pipeline runs `allurectl watch -- <test command>` and not the raw test command |
| Run started from Allure TestOps creates a new launch instead of using the existing one | `ALLURE_JOB_RUN_ID` does not reach `allurectl`. Check the declaration and the pass-through shown in the pipeline example for the CI |
| GitHub returns 422 when Allure TestOps starts the workflow | `workflow_dispatch` is missing, service inputs are not declared, or the Allure TestOps job has a parameter that is not a workflow input. The job was not updated from the build server after the workflow changed |
| All tests run although Allure TestOps selected only some | The test framework adapter does not read the test plan, or the test plan path is not visible to the test process, or the test command is not wrapped by `allurectl watch` |
| `failed to scan err pipe: bufio.Scanner: token too long` | Reduce test output to stdout or raise `--stdout-max-buff-size` |
| Certificate errors | Ask the user whether the instance uses a self-signed certificate before adding `--insecure` |

The steps on the Allure TestOps side (adding the CI integration, configuring the job, environment mapping) are done by the user in the Allure TestOps UI. Point them to the setup page of their CI system and list what has to be done there.

## When to send the user to support

Send the user to https://help.qameta.io and stop working on that part of the task when:

- several launches are needed from one CI build;
- a full integration (links to builds, start and rerun from Allure TestOps) is needed for a CI system that is not in the supported list;
- uploads fail and `--log debug` output does not point to a configuration problem you can fix in the pipeline;
- the required behaviour is not described in `allurectl --help` or in the public documentation.

Suggest attaching the `allurectl` version (`allurectl --version`), the pipeline definition and the `--log debug` output with the token removed.
