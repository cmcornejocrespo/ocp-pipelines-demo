# Red Hat OpenShift Pipelines

This is a sample tutorial to shows the main capabilities of Red Hat
OpenShift Pipelines (based in [Tekton](https://tekton.dev/)).

This tutorial walks through from the main components of this new way to
define CICD pipelines.

To follow it, the next requirements must be resolved:

* Red Hat OpenShift 4
* Red Hat OpenShift Pipelines operator
* [Tekton CLI](https://github.com/tektoncd/cli) (`tkn`)

If you don't have available a Red Hat OpenShift cluster, you could use
[Red Hat CodeReady Containers](https://github.com/code-ready/crc) to have
OpenShift 4 on your laptop.

**TIP**: Create a new project (e.g.) `pipelines-demo` to follow this repo:

```shell
oc new-project pipelines-demo
```

## Tasks

A `Task` is a collection of `Steps` that you define and arrange in a specific
order of execution as part of your continuous integration flow. A `Task` executes
as a Pod on your OpenShift cluster. A `Task` is available within a specific
namespace, while a `ClusterTask` is available across the entire cluster.

References:

* [TektonCD - Tasks](https://tekton.dev/docs/pipelines/tasks/)

### Sample Task

A sample task with a single step looks like similar to:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-task
spec:
  steps:
    - name: say-hello
      image: registry.redhat.io/ubi7/ubi-minimal
      command:
        - /bin/bash
      args: ['-c', 'echo Hello World']
```

Create this task in OpenShift:

```shell
oc apply -f 01-hello-task.yaml
```

Start the task and show the log output:

```shell
❯ tkn task start hello-task --showlog
TaskRun started: hello-task-run-gng4t
Waiting for logs to be available...
[say-hello] Hello World
```

### Parametrized Task

`Tasks` can also take parameters. This way, you can pass various flags to be
used in the `Task`. These parameters can be instrumental in making your `Tasks`
more generic and reusable across `Pipelines`.

A extended version of the previous task to ask for the name of a person looks
like similar to:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-params-task
spec:
  params:
    - name: person
      description: Name of person to greet
      default: World
      type: string
  steps:
    - name: say-hello
      image: registry.redhat.io/ubi7/ubi-minimal
      command:
        - /bin/bash
      args: ['-c', 'echo Hello $(params.person)']
```

This parameter could define a default value, in case it is not defined when the
task is started.

Create the task:

```shell
oc apply -f 02-hello-params-task.yaml
```

Start the task without parameters it will force to declare them:

```shell
❯ tkn task start hello-params-task --showlog
? Value for param `person` of type `string`? (Default is `World`) World
TaskRun started: hello-params-task-run-xpgv7
Waiting for logs to be available...
[say-hello] Hello World
```

`tkn` includes the `--use-param-defaults` argument to use the default values
of the parameters. The argument `--param` or `-p` sets up a different value for
the parameter:

```shell
❯ tkn task start hello-params-task \
  -p person="My name" \
  --showlog
TaskRun started: hello-params-task-run-jqfbs
Waiting for logs to be available...
[say-hello] My name
```

### Multiple Stepped Task

`Tasks` can have more than one `step`, allowing to specialize the task with more
detailed steps. The steps will run in the order in which they are defined in the
steps array.

The [03-hello-multisteps-task.yaml](./03-hello-multisteps-task.yaml) includes a
task with two steps to combine the execution of the task.

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-multisteps-task
  annotations:
    description: |
      A simple task with some steps that prints the a
      variable greeting message
spec:
  params:
    - name: person
      description: Name of person to greet
      default: World
      type: string
  steps:
    - name: write-hello
      image: registry.redhat.io/ubi8/ubi-minimal
      script: |
        #!/usr/bin/env bash
        echo Preparing greeting
        echo -n Hello $(params.person) > /tekton/home/hello.txt
        sleep 2
        echo Done!
    - name: say-hello
      image: node:14
      script: |
        #!/usr/bin/env node
        let fs = require("fs");
        let file = "/tekton/home/hello.txt";
        let fileContent = fs.readFileSync(file).toString();
        console.log(fileContent);
```

Create the task:

```shell
oc apply -f 03-hello-multisteps-task.yaml
```

Start the task.

```shell
❯ tkn task start hello-multisteps-task \
  -p person="My name" \
  --showlog
TaskRun started: hello-multisteps-task-run-xq79n
Waiting for logs to be available...
[write-hello] Preparing greeting
[write-hello] Done!

[say-hello] Hello My name
[say-hello] 
```

OpenShift has a dashboard to check and review the current status of the `Tasks` and `TaskRuns` from
the menu `Pipelines -> Tasks`.

![Tasks Run Dashboard](./img/tasksrun-dashboard.png)

## Pipelines

A `Pipeline` is a collection of `Tasks` that you define and arrange in a
specific order of execution as part of your continuous integration flow.
Each `Task` in a `Pipeline` executes as a `Pod` on your OpenShift cluster.
You can configure various execution conditions to fit your business needs.

In fact, tasks should do one single thing so you can reuse them across
pipelines or even within a single pipeline.

References:

* [TektonCD - Pipelines](https://tekton.dev/docs/pipelines/pipelines/)

### Sample Pipeline

A simple pipeline looks like similar to:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: say-things-pipeline
spec:
  tasks:
    - name: first-task
      params:
        - name: pause-duration
          value: "2"
        - name: say-what
          value: "Hello, this is the first task"
      taskRef:
        name: say-something-task
    - name: second-task
      params:
        - name: say-what
          value: "And this is the second task"
      taskRef:
        name: say-something-task
```

Create the task and pipeline:

```shell
oc apply -f 04-say-something-task.yaml
oc apply -f 04-say-things-pipeline.yaml
```

To start the pipeline:

```shell
❯ tkn pipeline start say-things-pipeline --showlog
PipelineRun started: say-things-pipeline-run-4wsvb
Waiting for logs to be available...
[second-task : say-it] And this is the second task
[first-task : say-it] Hello, this is the first task
```

Or create a `PipelineRun` definition to start the pipeline:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: say-things-pipelinerun
spec:
  pipelineRef:
    name: say-things-pipeline
```

```shell
oc apply -f 04-say-things-pipelinerun.yaml
```

OpenShift has a dashboard to check and review the current status of the `Pipeline` and `PipelineRun` from
the menu `Pipelines -> Pipelines`.

![Pipeline Run Dashboard](./img/pipelinesrun-dashboard.png)

### Parallel Pipeline

`Tasks` will be executed in the order defined in the pipeline, or create a
sequence using `runAfter` definition.

[05-say-things-in-order-pipeline.yaml](./05-say-things-in-order-pipeline.yaml) file
has a sample of order and parallel tasks in a pipeline.

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: say-things-in-order-pipeline
  annotations:
    description: |
      Sample pipeline saying things in different moments.
spec:
  tasks:
    - name: first-task
      params:
        - name: pause-duration
          value: "2"
        - name: say-what
          value: "Hello, this is the first task"
      taskRef:
        name: say-something-task
    - name: second-parallel-task
      params:
        - name: say-what
          value: "Happening after task 1, in parallel with task 3"
        - name: pause-duration
          value: "2"
      taskRef:
        name: say-something-task
      runAfter:
        - first-task
    - name: third-parallel-task
      params:
        - name: say-what
          value: "Happening after task 1, in parallel with task 2"
        - name: pause-duration
          value: "1"
      taskRef:
        name: say-something-task
      runAfter:
        - first-task
    - name: fourth-task
      params:
        - name: say-what
          value: "Happening after task 2 and 3"
      taskRef:
        name: say-something-task
      runAfter:
        - second-parallel-task
        - third-parallel-task
```

```shell
oc apply -f 05-say-things-in-order-pipeline.yaml
```

To start the pipeline:

```shell
❯ tkn pipeline start say-things-in-order-pipeline --showlog
PipelineRun started: say-things-in-order-pipeline-run-w6fvn
Waiting for logs to be available...
[first-task : say-it] Hello, this is the first task

[second-parallel-task : say-it] Happening after task 1, in parallel with task 3
[third-parallel-task : say-it] Happening after task 1, in parallel with task 2

[fourth-task : say-it] Happening after task 2 and 3
```

And the graphical representation of this pipeline run is:

![Parallel pipeline run](./img/say-things-in-order-pipelinerun.png)

### Workspaces

`Workspaces` allow `Tasks` to declare parts of the filesystem that need
to be provided at runtime by `TaskRuns`. A `TaskRun` can make these parts
of the filesystem available in many ways:

* using a read-only `ConfigMap` or `Secret`,
* an existing `PersistentVolumeClaim` shared with other `Tasks`, create
a `PersistentVolumeClaim` from a provided `VolumeClaimTemplate`,
* or simply an `emptyDir` that is discarded when the `TaskRun` completes.

`Workspaces` are similar to `Volumes` except that they allow a `Task` author
to defer to users and their `TaskRuns` when deciding which class of
storage to use.

The main use cases for `Workspaces` are:

* Storage of inputs and/or outputs
* Sharing data among `Tasks`
* Mount points for configurations held in `Secrets` or `ConfigMaps`
* A cache of build artifacts that speed up jobs

References:

* [TektonCD - Workspaces](https://tekton.dev/docs/pipelines/workspaces/)

This is a sample task with a workspace:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: count-files-workspace-task
spec:
  workspaces:
    - name: source
      description: The workspace consisting of repository.
  steps:
    - name: count
      image: registry.redhat.io/ubi8/ubi-minimal
      command:
        - /bin/bash
      args: ['-c', 'echo $(find /workspace/source -type f | wc -l) files in repo']
```

[07-count-workspace-pipeline.yaml](./07-count-workspace-pipeline.yaml) file describes
a sample pipeline using a workspace across different tasks. This example uses a
`ClusterTask` to demonstrate how to reuse these objects in a pipeline.

```yaml
---
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: count-workspace-pipeline
  annotations:
    description: |
      Sample pipeline using a workspace to share data between
      different tasks.
spec:
  params:
    - name: GIT_URL
      type: string
      default: "https://github.com/spring-projects/spring-petclinic.git"
    - name: GIT_REVISION
      type: string
      default: "main"      
  workspaces:
    - name: workspace
  tasks:
    - name: fetch-repository
      params:
        - name: url
          value: $(params.GIT_URL)
        - name: revision
          value: $(params.GIT_REVISION)
      taskRef:
        kind: ClusterTask
        name: git-clone
      workspaces:
        - name: output
          workspace: workspace
    - name: count-repo
      taskRef:
        name: count-files-workspace-task
      runAfter:
        - fetch-repository
      workspaces:
        - name: source
          workspace: workspace
```


Create the task and the pipeline:

```shell
oc apply -f 07-count-files-workspace-task.yaml
oc apply -f 07-count-workspace-pipeline.yaml
```

As this workspace requires a storage, we need to create the `PersistentVolumeClaim`.
The [07-workspace-pvc.yaml](./07-workspace-pvc.yaml) defines it.

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: workspace-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeMode: Filesystem
```

Create PVC to store the data:

```shell
oc apply -f 07-workspace-pvc.yaml
```

Start the pipeline requires to declare the workspace to use, among other
parameters declared in the pipeline:

```shell
tkn pipeline start count-workspace-pipeline \
    --param GIT_URL=https://github.com/spring-projects/spring-petclinic.git \
    --param GIT_REVISION=master \
    --workspace name=workspace,claimName=workspace-pvc \
    --showlog
```

Start again the pipeline with other parameters:

```shell
tkn pipeline start count-workspace-pipeline \
    --param GIT_URL=https://github.com/spring-projects/spring-boot.git \
    --use-param-defaults \
    --workspace name=workspace,claimName=workspace-pvc \
    --showlog
```

## Triggers

Tekton Triggers is a Tekton component that allows you to detect and extract
information from events from a variety of sources and deterministically instantiate
and execute `TaskRuns` and `PipelineRuns` based on that information.
Tekton Triggers can also pass information extracted from events directly to
`TaskRuns` and `PipelineRuns`.

References:

* [TektonCD - Triggers](https://tekton.dev/docs/triggers/)

To show how triggers works, we will extend our previous pipeline to be executed
with a trigger when a new change is pushed into the GitHub repository.

The main objects related with triggers are:

* `EventListener`: listens for events at a specified port on your OpenShift
cluster. Specifies one or more `Triggers` or `TriggerTemplates`.

For our use case we need to create an `EventListener` that will use a `TriggerTemplate`.
The [08-count-workspace-pipeline-eventlistener.yaml](./08-count-workspace-pipeline-eventlistener.yaml) defines it.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: count-workspace-pipeline-eventlistener
spec:
  serviceAccountName: pipeline
  triggers:
    - triggerRef: github-listener-trigger
```

Create a `EventListener` that uses a `Trigger`:

```shell
oc apply -f 08-count-workspace-pipeline-eventlistener.yaml
```

The `EventListener` will create a service to be used to access to. If we want
to use this service externally, we need to expose as a route:

```shell
❯ oc get svc
NAME                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
el-count-workspace-pipeline-eventlistener   ClusterIP   172.30.134.93   <none>        8080/TCP,9000/TCP   13m
❯ oc expose svc el-count-workspace-pipeline-eventlistener
```

The new should be similar to:

```shell
❯ oc get route
NAME                                        HOST/PORT                                                                                                   PATH   SERVICES                                    PORT            TERMINATION   WILDCARD
el-count-workspace-pipeline-eventlistener   el-count-workspace-pipeline-eventlistener-pipelines-demo.apps.cluster-76lkr.76lkr.sandbox1545.opentlc.com          el-count-workspace-pipeline-eventlistener   http-listener                 None
```

This command will get right url to use in the WebHook:

```shell
echo "$(oc get route el-count-workspace-pipeline-eventlistener --template='http://{{.spec.host}}')/hooks"
```

We will use this route later to integrate in our GitHub repository as a WebHook.

* `Trigger`: specifies what happens when the `EventListener` detects an event.
A `Trigger` specifies a `TriggerTemplate`, a `TriggerBinding`, and
optionally, an Interceptor.

Let's create a `Trigger` that will intercept an event (Github webook) and will execute a `TriggerTemplate`. Ir order to parse the payload and share that information to the `TriggerTemplate` a `TriggerBinding` is required.

The [08-count-workspace-pipeline-trigger.yaml](./08-count-workspace-pipeline-trigger.yaml) defines it.

```yaml
---
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: github-listener-trigger
spec:
  interceptors:
    - ref:
        name: github
      params:
        - name: secretRef
          value:
            secretName: github-interceptor-webhook
            secretKey: secret
        - name: eventTypes
          value: ["push"]
    - ref:
        name: cel
      params:
        - name: "filter"
          value: body.ref == 'refs/heads/main'
  bindings:
    - ref: count-workspace-pipeline-triggerbinding
  template:
    ref: count-workspace-pipeline-triggertemplate
```

Create a `Trigger` that uses a `TriggerTemplate`:

```shell
oc apply -f 08-count-workspace-pipeline-trigger.yaml
```

The trigger includes an [Interceptor](https://tekton.dev/docs/triggers/interceptors/),
that it is a "catch-all" event processor to perform payload filtering, to get
the details from GitHub repo.

```yaml
  interceptors:
    - ref:
        name: github
      params:
        - name: secretRef
          value:
            secretName: github-interceptor-webhook
            secretKey: secret
```

This secret will be used to add a security check to confirm that GitHub is
invoking the `EventListener` under a security context.

We need to create a `Secret` with the value to use from GitHub WebHook
to create a secured call.

```shell
oc apply -f 08-github-interceptor-webhook-secret.yaml
```

* `TriggerTemplate`: specifies a blueprint for the resource, such as a `TaskRun`
or `PipelineRun`, that you want to instantiate and/or execute when your
`EventListener` detects an event. It exposes parameters that you can use
anywhere within your resource’s template.

The [08-count-workspace-pipeline-triggertemplate.yaml](./08-count-workspace-pipeline-triggertemplate.yaml) defines it.

```yaml
---
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: count-workspace-pipeline-triggertemplate
spec:
  params:
    - name: git_url
      description: The git repository that hosts context  
    - name: git_revision
      description: The revision of the repository  
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: count-workspace-pipeline-run-
        annotations:
          tekton.dev/gitURL: "$(tt.params.git_url)"      
      spec:
        pipelineRef:
          name: count-workspace-pipeline
        params:
          - name: GIT_URL
            value: $(tt.params.git_url)
          - name: GIT_REVISION
            value: $(tt.params.git_revision)
        workspaces:
          - name: workspace
            persistentVolumeClaim:
              claimName: workspace-pvc
```

Create the `TriggerTemplate`:

```shell
oc apply -f 08-count-workspace-pipeline-triggertemplate.yaml
```

* `TriggerBinding`: specifies the fields in the event payload from which you
want to extract data and the fields in your corresponding `TriggerTemplate`
to populate with the extracted values. You can then use the populated fields
in the `TriggerTemplate` to populate fields in the associated `TaskRun` or `PipelineRun`.

The [08-count-workspace-pipeline-triggerbinding.yaml](./08-count-workspace-pipeline-triggerbinding.yaml) defines it.

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: count-workspace-pipeline-triggerbinding
spec:
  params:
  - name: git_url
    value: $(body.repository.url)
  - name: git_revision
    value: $(body.repository.default_branch)

```

Create the `TriggerBinding`:

```shell
oc apply -f 08-count-workspace-pipeline-triggerbinding.yaml
```

The latest step is create a new GitHub WebHook in our repository using the
route exposed above and adding the `/hooks` path. 

In your GitHub repo go to `Settings -> Webhooks` and click `Add Webhook`. The
fields we need to set are:

* **Payload URL**: Your external FQDN address from the route with `/hooks` path
* **Content type**: `application/json`
* **Secret**: Value defined in `github-interceptor-webhook` secret.

From now every time you push a new change in your repository, a new pipeline
execution will happen.

```shell
git push origin main
```

For more details about how to create a webhook, please, review this
[doc](https://docs.github.com/en/developers/webhooks-and-events/webhooks/creating-webhooks).

If you want to integrate with other Git server, you could use Webhooks, following the
examples created [here](./09-webhook)

## ... and beyond

This repo includes a short summary of many of the main objects and capabilities of
Red Hat OpenShift Pipelines (a.k.a. Tekton), but it is only a surface of all the them.
Please, go to the resources of this amazing project to learn and improve your
CICD pipelines in a cloud native way.

* [Red Hat OpenShift Pipelines Release Notes](https://docs.openshift.com/container-platform/4.10/cicd/pipelines/op-release-notes.html)
* [TektonCD - Docs](https://tekton.dev/docs/)
* [Tekton Playground](https://katacoda.com/tektoncd/scenarios/playground)
