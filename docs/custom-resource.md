---
slug: '/'
sidebar_label: 'Continuous Resource'
---

Custom Resources are created by using the tasktemplateV1.yaml file and are submitted to the Kubernetes Cluster using standard deployment tools.
The resources are stored in the Kubernetes cluster as instances of the Continuous Custom Resource Definition (CRD) created during the deployment of the Continuous Task Controller.
There is one Continuous Custom Resource Definition per Kubernetes cluster.

A Custom Resource is an object stored in the API server. It does nothing by itself. It records:
- spec: what you want, i.e. the workflow it belongs to and the Job template (containers, completions, parallelism, retries) to run when the host system starts the task.
- status: reserved fields (phase, jobName, logs, completedAt). The controller does not currently write them; the state of a running task is available through the controller's `taskstatus` API.

Once a Custom Resource has been submitted to the Kubernetes cluster using an appropriate deployment method, the Continuous Task Controller is 
informed by a Kubernetes Event that a new Custom Resource has been received by the Continuous Custom Resource Definition. 

There are three event types that are forwarded to the Continuous Task Controller by the Kubernetes environment. What the controller does with them depends on the configured host system (**HOSTSYSTEM**). The actions below apply to **OPCON**. With **NONE** the events are only logged, and **CONTINUUM** is not yet implemented (events are logged and skipped).

| Event | Fires When | What the Continuous Task Controller does (OPCON) |
|---|---|---|
| Added    | The resource is created. It also fires for every existing resource when the watch starts or reconnects. | Exports the job to OpCon. This creates the job with a POST, or updates it with a PUT if it already exists. |
| Modified | The resource's spec, metadata or status changes. | Handled the same as Added, so it updates the job with a PUT. |
| Deleted  | The resource is removed. | Disables the OpCon job (sets DisableBuild = true). The job is not deleted from the host system. If the disable fails it is not retried and the job must be disabled in OpCon manually. |

An event is skipped (and logged) if the required host settings below are missing, or if the resource has no `spec.workflowName`. A failed Added or Modified export is logged and retried the next time the watch re-lists the resources.

During the Continuous Task Controller deployment process, the target host system is configured using environment variables.

| Name | Description |
|---|---|
| **HOSTSYSTEM**           | Defines the target host system type - values are NONE, OPCON, CONTINUUM (default NONE) |
| **HOSTADDRESS**          | Defines the target host system Rest-API address - value address:port (https is assumed unless a scheme is given) |
| **HOSTINSECURE**         | Default false. When true, the host system's HTTPS certificate is not validated and HOSTTOKEN is sent to whatever server answers at HOSTADDRESS. For development use only. |

For an OPCON host system, the OpCon Rest-API is used to create / modify the task definition and the Rest-API connection to the OpCon system needs to be defined including
certificate information.

The following environment variables are used to define an OpCon Host System.

| Name | Description |
|---|---|
| **HOSTTOKEN**            | The token is an application token used for authentication when conversing with the OpCon Rest-API |
| **HOSTAGENT**            | The name of the OpCon ACS "Continuous Task" machine that runs the jobs. It is set as the primary machine of each job created. |
| **HOSTPROPERTYPREFIXES** | Optional comma-separated, case-sensitive list of prefixes. Environment variable names or values may contain OpCon property tokens (`[[name]]`) only if the property name starts with one of these prefixes. For example, with `CT_` only `[[CT_...]]` properties can be used. The default is empty, which allows no tokens. A resource that references a property not allowed by the list is not exported, and an existing OpCon job keeps its previous definition. |

If the workflow contained in the workflowName attribute does not exist, the workflow will be created and the task will then be
associated with the created workflow. An existing workflow is used as it is. Workflow names are not case-sensitive.

The spec:workflowName is used as the name of the workflow and the metadata:namespace and metadata:name are used to create the job name (format **namespace.name**; names longer than 64 characters are truncated and given a short hash suffix).
A default **CT_AllDays** frequency (every day) is used when creating the OpCon job, starting at 00:00. No dependencies are included.

The only information passed to the OpCon job definition is 
- from the metadata section the namespace and name attributes are included in the OpCon definition. When OpCon runs the job, the agent uses them to ask the controller (`POST /continuoustc/api/taskstart`) to start the task.
- all environment variables and their literal values are included in the OpCon definition. 


Custom Resource example

```

apiVersion: "continuous.com/v1"
kind: ContinuousTask
metadata:
  namespace: continuous-system
  name: task-t003
spec:
  workflowName: "workflow-name"
  template:
    spec:
      completions: 5
      parallelism: 2
      ttlSecondsAfterFinished: 300
      backoffLimit: 0
      restartPolicy: Never
      containers:
      - name: ctaskt003
        image: java:11.5
        command:
        - java
        args:
        - -jar
        - /app/jars/javademo.jar
        env:
        - name: JOBID
          value: "123456"
        - name: EVCODE
          value: "0"
        - name: EVTIME
          value: "10"

```

where
     apiVersion              must be **"continuous.com/v1"**.
     kind                    must be **ContinuousTask**.

     metadata
       namespace             is the namespace where the Kubernetes Job must execute within.
       name                  is the name of the task, which must be unique within each namespace of the Kubernetes Cluster.

     spec
       workflowName          (required, not empty) is the workflow name of the target system where the task should be inserted into or updated.
       template
          spec
             completions     the number of times to execute the task.
             parallelism     how many tasks can run simultaneously.
             ttlSecondsAfterFinished  how many seconds to retain the job after it has completed before destroying it.
             backoffLimit    how many times to retry the job if it fails
             restartPolicy   values are
                             OnFailure Only restarts the container if it exits with an error (non-zero exit status).
                             Never     Does not automatically restart the terminated container.
                             (Always is not valid for a Kubernetes Job.)
             containers      (required) can enter multiple containers (each container starts with a -)
             - name          (required) the name of the container
               image         (required) the image to use when starting the job
               command       list of the command and its parts to pass to the job (optional)
               args          list of arguments to pass to the command (optional)
               workingDirectory  working directory of the container (optional)
               env           environment variables to process (optional)
               - name        variable name
                 value       variable value, always a quoted string
               ports, volumeMounts, volumeDevices   optional, as for a Kubernetes container
