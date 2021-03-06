# ABCD (Application for Big Computational Data) Specification (v1.0)

## Background

Scientists routinelty use applications to compute and perform analyses either using personal computers (local hardware) or remote resources (distal hardware, such as high performance clusters or cloud systems). Applications can be submitted through a varity of interfaces such as Python, R or MatLab scripts, compiled c code, or *bash terminal scripts*. Each application has unique requirements for computing and software libraries dependencies. Furthermore, types of data and input files differ from application to application. In order, to use applications developed from other investigaotrs using a new hardware system or a new software environment to reproduce an analysis each user must take care of isntalling the application software and all associated dependencies required by the software, this is in addition to preparing appropriate input files formats and organzie the file structure in the way expected by the application. This process is a substantial budern and limits reuse of scientific software and code.

### Application Abstraction

To mitigate this problem, developers have recently started containerising their applications using containerization methods, such those provided by docker. Containerization can reduce the complexity involved with installing applications and avoid dependency conflicts between different applications. This approach, however, still leaves the major task of preparing the appropriate input files for each application to the end user.

### Data Format Abstraction

Recently, there has been a proposal to create a standard data structures (such as BIDS for Neuroscience) and create containers (such as BIDS Apps) that are designed to receive input files in a common data structure. Such approach can greatly reduce the overhead of preparing the input files, and foster truly reusable software framework within a specific domain of science.

### Execution Abstraction

These advancements greatly benefit advanced domain experts, however, we need yet another layer of abstraction; commonly refer to as a *workflow management*, which is necessary to orchestrate a workflow required to process large amounts of data, or to make the applications accessible to novice users through an interface such as web portals. 

Indeed, many workflow management systems have been developed, but so far there has not been a widely adopted generic specification describing how to start, stop, and monitor applications so that applications can be programmatically executed across different workflow managers. This necessitates developers to create applications (or the application wrappers) for each workflow managers making them difficult to reuse. 

## GOAL

This specification proposes a very simple standard to allow abstraction of application execution and monitoring in order to make it easier to incooporate compliant applications into various workflow management systems.

Please note that, in this specification, *workflow management system* can be as simple as a small shell script that manages execution of a small and static set of applications, or a large software suite that can handle asynchronous executions of multi-user / multi-cluster workflows with smart monitoring system for reliablity and error-torrelence.

## package.json

All ABCD compliant application must have `package.json` under the root directory of the application (such as root folder of a git repo) This file can be created by hand, or by using [`npm init`](https://docs.npmjs.com/cli/init) command (npm is a commonly used package manager for nodejs - but the application itself does not have to be a node app).

```json
{
  "name": "abcd-hello",
  "version": "1.0.0",
  "description": "sample ABCD app",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "Soichi Hayashi <hayashis@iu.edu>",
  "license": "MIT"
}
```

`package.json` describes basic information about your application (name, description, license, dependencies, etc..). An important section is the `"scripts"` section which lists supported `application hooks`. 

## Application Hooks

Applicatoin hooks are special scripts that are meant to be executed by the workflow manager (but it can also be executed locally to test your application).

At minimum, most workflow management systems must be able to.. 

* Start your application (or run it synchronously if it's a very simple application), 
* Stop your application - based on user request or other reasons
* Monitor the status of your application, and know if it is completed / failed / running, etc..

ABCD compliant workflow management system must publish list of these hooks inside `package.json` like following..

```json
  "scripts": {
    "start": "start.sh",
    "stop": "stop.sh",
    "status": "status.sh"
  },
 ```

In this example, this application tells ABCD compliant workflow management system that we have 3 shell scripts on a root directory of this application to `start`, `stop`, and `monitor` status of this application. Each hook can point to any script, or executables. For example, "start", could be mapped to `start.py` (instead of `start.sh`) if you prefer to use Python, or any binary executable, or even a command line such as `"qsub start.sub"`.

All hook scripts must be executable (`chmod +x start.sh`)

### `start` Hook

`start.sh` (or any script or binary that you've mapped "start" to) must *start* the application - but not actually *run* it. Meaning, you must nohup and background a new process (using & at the end of the command line) that actually do the processing. For example...

```bash
#!/bin/bash
nohup matlab -r $SCA_SERVICE_DIR/main &
ret=$?
echo $! > run.pid
exit $ret
```

If you are spawning your program like above, you should store the PID of your application so that later you can use to monitor the process or stop it (explained later).

Or, for PBS cluster, you can simply do

```bash
#!/bin/bash
qsub myapp.sub > jobid
exit $?
```

qsub simply submits the job to the local cluster and exits (it does not *run* your application), so you don't have to nohup / background the process.

If you have a very simple application that always finishes in a matter of seconds, you may actually run the application itself inside start.sh and block before exiting, but all start.sh is expected to complete within a few seconds. Otherwise you will risk your application to be be killed by the workflow management system - depending on its implementation.

#### Exit code

Start script should return exit code 0 for successful start or submission of your application, and non-0 for startup failure.

#### stdout/stderr
Start script can output startup log to stdout, or any error messages to stderr. Such messages to be handled by the workflow manager and relayed to the user in appropriate manner.

### `status` Hook

This script will be executed by workflow manager periodically (every few minutes or longer) to gather status about the application. You can simply check for the PID of the application to make sure that it's running, or for a batch system, you can use `qstat` / `condor_status` type command to query for your application. 

Here is an example of job status checker for PBS Jobs > [sca-service-dtinit](https://github.com/soichih/sca-service-dtiinit/blob/master/status.sh)

#### Exit Code

Status script should return one of following code

* 0 - Job is still running
* 1 - Job has finished successfully
* 2 - Job has failed
* 3 - Job status is *temporarily* unknown (should be retried later)

Status script return within a few seconds, or workflow manager may assume that the job status is unknown.

#### stdout/stderr

Any detail about the current job status can be reported by outputting any messages to stdout. If your status script is smart enough to know the execution stage of your application, you could output a message such as "Job 25.5% complete - processing XYZ", for example. The message should be small, however, and usually be limited to a single string. Workflow manager may use it as a status label for your workflow to be displayed on UI.

### `stop` Hook

This script will be executed when workflow manager wants to stop your job for whatever the reason (maybe user hit CTRL+C on the simple workflow manager script, or user has requested termination of workflow via the web UI of the portal, etc..)

Here is an example stop script for PBS application

```bash
#!/bin/bash
qdel `cat jobid`
```

`jobid` is created by `start.sh` in this example. 

Or, your stop script may choose to do `kill -9 $pid`, or run application specific termination command like `docker stop $dockerid`.

#### Exit Code

Stop script should return one of following code.

* 0 - Application was cleanly terminated
* 1 - Failed to terminate the application cleanly.

If your stop script returns code 1, workflow manager may report back to the user that the termination has failed (and user may repeat the request), or some workflow manager may simply retry later on automatically.
ret=$?
## Environment Parameters

ABCD application will receive all standard ENV parameters set by users or the cluster administrator. ABCD workflow manager should also set following ENV parameters.

`$SCA_TASK_ID` Unique ID for this application instance. Usually a DB id for the application instance (or *task*) generated by the workflow manager.

`$SCA_TASK_DIR` Initial working directory for your application instance (not the application installation directory)

`$SCA_WORKFLOW_DIR` Directory where $SCA_TASK_DIR is stored (should be a parent of $SCA_TASK_DIR)

`$SCA_SERVICE` Name of the application executed. Often a github repo ID (like "soichih/sca-service-dtiinit")

`$SCA_SERVICE_DIR` Directory where the application is installed.

* "SCA" maybe renamed to something else in the future..

## Application Installation Directory

ABCD is designed for Big Data and parallel processing. On a highly parallel environment, workflow manager often stores your application in a common shared filesystem and share the same installation across all instances of your application (there could be several thousands of such instance running concurrently - like in OSG), although the working directory will be created for each instance to stage your input and output data. 

ABCD application will be executed with current directory set to this workfing directory; *not the directory where the application is installed*. This means that, in order to reference other program files in your application from another program file, you will need to prefix the location by the special environment parameter of `$SCA_SERVICE_DIR`. This ENV parameter will be set by all ABCD compliant workflow manager to inform your application where the application is installed.

For example, let's say you have following files in your application.

```
./package.json
./start.sh
./main.py
```

In your `start.sh`, even though `main.m` is located next to `start.sh`, your current working directory may be outside this application directory. Therefore, you will need to prefix `main.py` with `$SCA_SERVICE_DIR` inside `start.sh`, like..

```bash
#!/bin/bash
nohup python $SCA_SERVICE_DIR/main.py &
ret=$?
echo $! > run.pid
exit $?
```

### Development 

In earlier abouve `start.sh` example, if you are running it locally for development purpose, most likely `$SCA_SERVICE_DIR` is not set, so it will try to reference a path `/main.py` which does not exist. In order to help debugging your application on a local directory, you'd like to add something like following at the top of your `start.sh`

```
if [ -z $SCA_SERVICE_DIR ]; then export SCA_SERVICE_DIR=`pwd`; fi
```

This will allow your application to be executable on the same directory where your current directory is. Please see [https://github.com/soichih/sca-service-dtiinit/blob/master/start.sh] for more concrete example.

Obviously, if you have no other files than the actual hook scripts themselves (maybe you are just running a docker container from `start.sh`), you don't need to worry about this issue. 

## Input Parameters (config.json)

For command line applications, input parameters can be passed via command line parameters. In order to allow workflow manager to execute ABCD application, all input parameters must be passed in `config.json`. When users specify input parameters through various UI, workflow manager will pass that information by generating `config.json` containing all parameters used to execute the application. 

ABCD application must parse `config.json` within the application to pull any input parameter. Or, your `start.sh` can do the parsing and pass those parameters to your application as command line parameters or ENV parameter.

For example, let's say user has specified `input_dwi` to be `"/N/dc2/somewhere/test.dwi"` and `paramA` to be `1234` on some UI. Workflow manager will construct following `config.json` and stores it in a working directory (on the current directory) prior to application execution.

```json
{
 "input_dwi": "/N/dc2/somewhere/test.dwi",
 "paramA": 1234
}
```

Your application can parse this directly, or in your `start.sh` you can do something like following to *pass* input arguments to your application.

```bash
#!/bin/bash
docker run -ti --rm \
-v `jq '.input_dwi[] config.json'`:/input/input.dwi \
someapp paramA=`jq '.paramA[]' config.json` \
> dockerid
```

`jq`[https://stedolan.github.io/jq/] is a popular JSON parsing tool for command line.

ABCD specification currently does not allow defining a valid input parameters that can be used in the config.json for each application. Such definition could be stored as part of `package.json` in the future to allow auto generation of the application submission UI. We could also borrow specification from a system such as [GenApp](https://cwiki.apache.org/confluence/display/AIRAVATA/GenApp)

## Input Files

Staging of input files are outside the scope of ABCD specification. It is a task for workflow manager to transfer / stage any necessary input files (referenced in the `config.json`) prior to executing your application.

## Output Files

Your application can output any output data, or log files in the current working directory.  The structure of the output files is up to each application. To pass the output data to subsequent applications, `config.json` can be constructed to reference those output files as inputs to latter application.

It is customary, but not required, to generate a file named `products.json` in the working directory along with any actual output files with a list of all output files generated in order for workflow manager to parse and stores catalog of the output files generated by your application instance. Such information maybe useful, for example, if your application can generate unpredicatble amount of graphics based on your input files and you'd like to display them on a workflow GUI.

## ABCD Referemce Application Implementations

List of ABCD compliant services can be found here [https://github.com/soichih?tab=repositories&q=sca-service]

## ABCD Reference Workflow Manager Implementation

Currently, the sca-wf is the only workflow manager that uses this specification, and can be used as a reference implementation. [https://github.com/soichih/sca-wf]


