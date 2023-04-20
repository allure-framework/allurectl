# allurectl cookbook

## Downloading allurectl

`allurectl` can be downloaded either manually and then used during the build or during the preparation stage of your build task.

In example below we download latest release linux application for linux OS with x64 CPU:

```bash
wget https://github.com/allure-framework/allurectl/releases/latest/download/allurectl_linux_amd64 -O ./allurectl
chmod +x ./allurectl
```

You need to select the binary file suitable for your OS and CPU architecture from [the available options](https://github.com/allure-framework/allurectl/releases/latest).

## allurectl working modes

There are two modes:

1. Non-CI mode
2. CI mode

After **allurectl** starts from the command line, it checks whether CI server specific variables are present in its context.

### Non-CI mode

If CI server specific variables are absent, the data upload is considered as manual upload from local PC (i.e. not from a CI pipeline).

Local upload does not allow using of launch parameters, job-run parameters.

### CI mode

If CI server specific variables are present, the data upload is considered as work in CI pipeline's context and this allows using of launch parameters, job-run parameters.

## Allure TestOps API token

Before you will be able to use `allurectl`, you need to generate Allure TestOps secret token in your profile, so `allurectl` will be able to be authenticated on Allure TestOps side.

The process of getting of the API token is described in [Allure TestOps documentation](https://docs.qameta.io/allure-testops/integrations/com/allure-token/).

## Passing the parameters to allurectl

### Environment variables

The following environment variables need to be set for easier usage of `allurectl`

| Env variable      | Comment                                                                                     |
|-------------------|---------------------------------------------------------------------------------------------|
| ALLURE_ENDPOINT   | URL of Allure TestOps server                                                                |
| ALLURE_PROJECT_ID | the ID of a project in Allure TestOps, this is the 1st column of Allure TestOps main screen |
| ALLURE_TOKEN      | the user's personal access token generated in the user's profile in section **API Tokens!** |
| ALLURE_LAUNCH_NAME| Name of the launch that will be displayed in Allure TestOps UI                              |

### Command line switches

> TBD

## Test connection

> This needs to be used for connection check purposes only. Do not use this command in your pipelines to authenticate the test results upload process.
> The authentication for test results upload routines is done otherwise (see below).

This could be done from your local computer or in a CI pipeline (see remark above).

To check the connection to Allure TestOps instance you can use the following command:

``` bash
export ALLURE_TOKEN=<API-TOKEN>
export ALLURE_ENDPOINT=https://demo.testops.cloud
$ allurectl auth login
```

Alternatively, you can use CL switches, but we recommend using the environment variables.

For more information use the `allurectl --help auth` command.

## Upload the test results to Allure TestOps

There are 2 options.

- usage of `watch` routine
- usage of `upload` routine

### watch

`watch` is main recommended workflow for test results upload.

Generally, **allurectl watch** does the same things which **allurectl upload** does with one important difference - **watch** command allows sending the test result in real time, i.e. you don't need to wait till all the tests are completed and that will decrease the workload and hasten the test results processing on Allure TestOps side.

**allurectl watch** is a wrapper for your test execution, i.e. you need to provide the command which you are using to execute your tests to **allurectl watch**.

#### The usage

As usual you need to provide **parameters** needed for **the connection to Allure TestOps** server (in the example below these are the environment variables, and we strongly recommend you using the environment variables to pass these parameters to allurectl), then you show allurectl where you're expecting the test results to appear and then provide the command to start tests execution.

```bash
export ALLURE_ENDPOINT=https://allure.company.com
export ALLURE_TOKEN=55555555-5555-5555-5555-555555555555
export ALLURE_PROJECT_ID=100
export ALLURE_LAUNCH_NAME="Hello, world" # you can use here the environment variables of your job/pipeline
...
allurectl watch --results path/to/allure-results -- ./gradlew clean test
```

alternatively, you can provide all the start parameters in the environment variables, so the **watch** will look prettier and concise:

```bash
export ALLURE_ENDPOINT=https://allure.company.com
export ALLURE_TOKEN=542dcd56-b0e2-4fdd-8ecf-bacf0f33d505
export ALLURE_PROJECT_ID=12
export ALLURE_LAUNCH_NAME="Hello, world"
export ALLURE_RESULTS=path/to/allure-results
...
allurectl watch -- ./gradlew clean test
```

### upload

> We recommend using allurectl **watch** to send the data from CI. Use upload only in case `watch` is not acceptable for you.

`upload` workflow is only applicable when the test results are available after all tests execution (there are such test frameworks, not many but they exist) or you need to upload a directory with the test results you have.

Use **allurectl** with the `upload` command **after** all your tests run. We do not recommend using allurectl upload as background process.

#### Upload using command line parameters

```bash
allurectl upload --endpoint https://allure.company.com \
    --token 55555555-5555-5555-5555-555555555555 \
    --project-id 100 \
    --launch-name "Local PC manual launch 2200-12-31" \
    path/to/allure-results
```

#### Upload using environment variables

```bash
# Define environment variables
export ALLURE_ENDPOINT=https://demo.testops.cloud
export ALLURE_TOKEN=55555555-5555-5555-5555-555555555555
export ALLURE_PROJECT_ID=100

# Run upload process somewhere
allurectl upload --launch-name "Local PC manual launch 2200-12-31" path/to/allure-results
```

Please refer to your [CI settings details]({{< relref "../integrations/ci-servers#the-workflow" >}}) to set up allurectl environments variables.

## Tests rerun and selective run with allurectl

The most important thing with test rerun and selective run is the **test plan**. In this particular case when we're talking about the selective run and rerun (rerun is actually is a special case for selective run) the **test plan** is a **file** (specifically, it's **testplan.json**) with the **list of test cases**, that your test framework needs to run.

Now, let's discuss how this integration works.

### Selective tests run integration

1. On Allure TestOps side we create a **list of test cases** we need to **(re)run** on CI side. Each test case is identified by AllureID and a selector. Selector is the way test framework identifies certain test, generally it is the combination of package name, class name and method name but can be test framework dependant.
2. The list of test cases is then saved to CI into **testplan.json**
3. On CI side we create an environment variable **ALLURE_TESTPLAN_PATH** and set the value to the **path to testplan.json** file
4. When your project with tests is being executed, then Allure TestOps integration with test framework checks if **ALLURE_TESTPLAN_PATH** is available
   1. If the variable **ALLURE_TESTPLAN_PATH** is available, the integration tries to find the file **testplan.json** using **ALLURE_TESTPLAN_PATH** value
   2. If the file **testplan.json** is successfully read, then the integration instructs the test framework to run only the tests specified in the **testplan.json**.

If CI starts all tests from your project, then either something is not configured on your side or your test framework has no integration with Allure TestOps.
If you are facing this, please file a support (not a bug) request to our [technical support.](https://help.qameta.io)

#### testplan.json structure

The testplan.json file looks like follows.

Knowing this structure, you can create testplan.json on your local PC, initialize the environment variable **ALLURE_TESTPLAN_PATH** with path to **testplan.json** and run your tests locally without any additional filters in the same session, if only tests from **testplan.json** will run, then you have the working integration for selective run, otherwise you need to configure the integration or [develop it for your test framework.](https://help.qameta.io)

```json

{
  "version":"1.0",
  "tests": [
    {
      "id": "10",
      "selector": "io.qameta.allure.examples.junit4.AllureSimpleTest.allureSimpleTest"
    },
    {
      "id": "11",
      "selector": "io.qameta.allure.examples.junit4.AllureParameterizedTest.allureParameterizedTest[First Name]"
    }
  ]
}

```

where

- **id** is the test ID (Allure ID) from Allure TestOps
- **selector** is the alternative ID which is equal to test's full path by default.
  - we're planning to extend the information provided in the selector
  ![if you see this write to support.qameta.io](./img/test-case-full-path-and-id.png)

### Saving the test plan to CI, run tests and upload the tests results

In all CIs we have the same sequence:

1. Allure TestOps starts a Job and adds **ALLURE_JOB_RUN_ID** = <id> to the job.
   - if allurectl detects **ALLURE_JOB_RUN_ID** then it ignores all other variables an session is created in the Job run set by Allure TestOps.
2. Allure TestOps provides the environment variable **ALLURE_TESTPLAN_PATH: ./testplan.json** to the job run.
3. **allurectl** creates **testplan.json** and fills it with the information about the list of the tests.
   - this is done using the command `allurectl job-run plan --output-file ${ALLURE_TESTPLAN_PATH}`
4. allurectl executes your tests and makes the index list of the test results files and send the tests results files to Allure TestOps using **watch** command
   - `allurectl watch -- ./gradlew clean test`

## Getting Allure TestOps launch information

### The task

We want to get the information on a launch created after the execution of `watch` or `upload` workflows to pass the information to chat, email message etc.

### How to
    
> This works in CI mode only    

The information on the entities created on Allure TestOps side can be placed to the environment variables and then used by invoking of the following sequence of the commands:

```shell
#define env vars
export ALLURE_ENDPOINT=https://demo.testops.cloud
export ALLURE_TOKEN=<toke>
export ALLURE_PROJECT_ID=<ID>
export ALLURE_LAUNCH_NAME="Hello, world"
export ALLURE_RESULTS=path/to/allure-results
# exec the tests
allurectl watch -- ./gradlew clean test

export $(allurectl job-run env)
# this will just show the list of all ENV vars related to allurectl execution
printenv | grep ALLURE_ 
```

`printenv | grep ALLURE_ ` will result in the following output

```shell
ALLURE_ENDPOINT=https://demo.testops.cloud
ALLURE_LAUNCH_ID=11111
ALLURE_RESULTS=./allure-results
ALLURE_JOB_RUN_ID=12345
ALLURE_LAUNCH_TAGS=master, gitlab, demo, pytest, skip-live-doc, ignore
ALLURE_TOKEN=[MASKED]
ALLURE_TESTPLAN_PATH=./testplan.json
ALLURE_LAUNCH_NAME=allure-pytest - 1ea04f48
ALLURE_JOB_RUN_URL=https://demo.testops.cloud/jobrun/14433
ALLURE_LAUNCH_URL=https://demo.testops.cloud/launch/31897
ALLURE_PROJECT_ID=433
```

To provide the link to the created launch you can use either `ALLURE_JOB_RUN_URL` or `ALLURE_LAUNCH_URL`.

`ALLURE_JOB_RUN_URL` is an entity (there could be N job-runs) inside a launch, so if you merge two or more launches in one, then `ALLURE_JOB_RUN_URL` will always point to a correct launch.
