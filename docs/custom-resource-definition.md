---
slug: '/'
sidebar_label: 'Continuous Custom Resource Definition'
---

A Custom Resource Definition (CRD) is a powerful Kubernetes object that allows you to define your own custom resource types and extend the Kubernetes API. Once you define a CRD, Kubernetes learns a new "word" and can treat objects of that new type just like its built-in ones. The ContinuousTask Controller is a separate deployment (one per cluster) that watches the custom resources cluster-wide; Kubernetes sends it events (Add, Modify, Delete) as resources change.  

It is a YAML file that defines your object to the Kubernetes API server. It includes the name (what it is called) and what type of information will be held in the custom resource definition.

## Continuous Custom Resource Definition
The resource definition is an extension to the Kubernetes Job definition creating a specific task definition that is supported by the ContinuousTask Controller.

The custom resource definition is applied to each Kubernetes Cluster by submitting a yaml file. Once the custom resource definition has been registered with the cluster it remains active until it is removed or the cluster has been destroyed.

When a Continuous Custom Resource is added, modified or removed, Kubernetes sends an event to the ContinuousTask Controller. The controller relays each event to the configured host system (OpCon or Continuum) with the resource namespace, name, `workflowName` and the environment variables declared on its containers.

These events are as follows:

- Add     a new custom resource was submitted to the custom resource definition.
- Modify  an existing custom resource was submitted to the custom resource definition.
- Delete  a custom resource was removed from the custom resource definition.

## Continuous Custom Resource
A Continuous Custom Resource is a Kubernetes Job definition that is submitted to the Continuous Custom Resource Definition. The resources (i.e. job definitions) are added to the custom resource by submitting yaml files (see **Continuous Task Template**). The task definition is stored in the custom resource definition and persisted by the Kubernetes Cluster.

A Continuous Custom Resource is executed when the host system sends a task start request (`POST /continuoustc/api/taskstart`) to the ContinuousTask Controller. The controller retrieves the resource by name, builds a Kubernetes Job from it, applies any environment variables supplied in the request over the resource's own (existing names are overridden, new names are appended), and submits the Job. Job state is returned through `GET /continuoustc/api/taskstatus/{id}`.

### Continuous Custom Resource format
A resource is a yaml file with the api/version ("continuous.com/v1"). `workflowName` and `template.spec.containers` (with `name` and `image`) are required; all other fields are optional.

```

apiVersion: "continuous.com/v1"
kind: ContinuousTask
metadata:
  namespace: (default)
  name: (the name of the task)
spec:
  workflowName: "(the name of the workflow on the target OpCon or Continuum system where the task should be created)"
  template:
    spec:
      completions: (integer - the number of times to execute the task)
      parallelism: (integer - the number of pods to run in parallel)
      ttlSecondsAfterFinished: (integer - seconds before a finished job is cleaned up, e.g. 300)
      backoffLimit: (integer - number of times to retry the task if it fails, e.g. 0)
      restartPolicy: (Never)
      containers:
      - name: (container name for the task)
        image: (the image to load)
        command: (list - the command to execute)
        args: (list - the arguments to pass to the command)
        workingDirectory: (optional working directory)
        env:
        - name: (variable name)
          value: "(variable value - always a quoted string)"
        ports: (optional list - containerPort, hostIP, hostPort, name, protocol)
        volumeMounts: (optional list - name, mountPath, subPath, subPathExpr, readOnly, mountPropagation, recursiveReadOnly)
        volumeDevices: (optional list - name, devicePath)

```

## Continuous Custom Resource Definition YAML

```

apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: continuoustasks.continuous.com
spec:
  group: continuous.com
  names:
    kind: ContinuousTask
    listKind: ContinuousTaskList
    plural: continuoustasks
    singular: continuoustask
  scope: Namespaced
  versions:
  - name: v1
    schema:
      openAPIV3Schema:
        properties:
          spec:
            properties:
              template:
                properties:
                  spec:
                    properties:
                      backoffLimit:
                        format: int32
                        type: integer
                      completions:
                        format: int32
                        type: integer
                      containers:
                        items:
                          properties:
                            args:
                              items:
                                type: string
                              nullable: true
                              type: array
                            command:
                              items:
                                type: string
                              nullable: true
                              type: array
                            env:
                              items:
                                properties:
                                  name:
                                    type: string
                                  value:
                                    nullable: true
                                    type: string
                                required:
                                - name
                                type: object
                              nullable: true
                              type: array
                            image:
                              type: string
                            name:
                              type: string
                            ports:
                              items:
                                properties:
                                  containerPort:
                                    format: int32
                                    type: integer
                                  hostIP:
                                    nullable: true
                                    type: string
                                  hostPort:
                                    format: int32
                                    nullable: true
                                    type: integer
                                  name:
                                    nullable: true
                                    type: string
                                  protocol:
                                    nullable: true
                                    type: string
                                type: object
                              nullable: true
                              type: array
                            volumeDevices:
                              items:
                                properties:
                                  devicePath:
                                    type: string
                                  name:
                                    type: string
                                required:
                                - devicePath
                                - name
                                type: object
                              nullable: true
                              type: array
                            volumeMounts:
                              items:
                                properties:
                                  mountPath:
                                    type: string
                                  mountPropagation:
                                    nullable: true
                                    type: string
                                  name:
                                    type: string
                                  readOnly:
                                    nullable: true
                                    type: boolean
                                  recursiveReadOnly:
                                    nullable: true
                                    type: string
                                  subPath:
                                    nullable: true
                                    type: string
                                  subPathExpr:
                                    nullable: true
                                    type: string
                                required:
                                - mountPath
                                - name
                                type: object
                              nullable: true
                              type: array
                            workingDirectory:
                              nullable: true
                              type: string
                          required:
                          - image
                          - name
                          type: object
                        type: array
                      parallelism:
                        format: int32
                        type: integer
                      restartPolicy:
                        type: string
                      ttlSecondsAfterFinished:
                        format: int32
                        nullable: true
                        type: integer
                    required:
                    - containers
                    type: object
                required:
                - spec
                type: object
              workflowName:
                minLength: 1
                type: string
            required:
            - template
            - workflowName
            type: object
          status:
            properties:
              completedAt:
                format: date-time
                nullable: true
                type: string
              jobName:
                nullable: true
                type: string
              logs:
                nullable: true
                type: string
              phase:
                type: string
            type: object
        required:
        - spec
        type: object
    served: true
    storage: true
    subresources:
      status: {}

```

## Continuous Task Template

```

apiVersion: "continuous.com/v1"
kind: ContinuousTask
metadata:
  namespace: continuous-system
  name: task-t003
spec:
  workflowName: "CT-INJECT"
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