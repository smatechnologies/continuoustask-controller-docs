---
slug: '/'
sidebar_label: 'Installation'
---


# Installation


## ContinuousTask Controller

.. download image from docker hub
.. download files from files.smatechnologies.com

### Configuration
The deployment of the Continuous Task Controller requires setting secrets, certificates and various environment variables. This process defines information on the target Continuous host system and 
certificates used to provide secure communication between the Continuous Host System and the Continuous Task Controller. 

These definitions are contained in the continuoustask-controller.yaml file which is used to deploy the Continuous Task Controller image.

#### Secrets and Certificates

The deployment of the Continuous Task Controller requires the creation of various secrets that contain the token value of the HOSTSYSTEM and the Certificates that provide secure communications between
the host system and Continuous Task Controller.

The scripts should be run after the namespace has been created, before the Continuous Task Controller is deployed.

##### HOSTTOKEN
The credential consists of an application token used to provide the OpCon host system Rest-API authentication. The deployment reads it from the **continuoustask-host-token** Secret (key HOSTTOKEN, optional), so the token never appears in the yaml file. The pod starts without the Secret, but the host system rejects the empty token. The controller reads the Secret only when it starts, so the scripts restart it by default.
The Secret value can be created by running the **install-hosttoken.ps1** or **install-hosttoken.sh** script.

```
Usage
cd .\install
.\install-hosttoken.ps1                # prompts for the token, then restarts the controller
.\install-hosttoken.ps1 -NoRestart     # set the Secret only


Usage
chmod +x install-hosttoken.sh
./install-hosttoken.sh                          # prompts silently for the token, then restarts the controller
./install-hosttoken.sh -R                       # set the Secret only
```

| Secret | Contents | Purpose |
|---|---|---|
| continuoustask-host-token | HOSTTOKEN | Contains the token value used to provide authentication to the Host OpCon system |

##### Certificates

The certificate information for the OpCon host connection and the Continuous Task Controller Executor Rest-API must be inserted into the required secrets before the Continuous Task Controller is deployed.

The OpCon host certificate is usually a self-signed certificate created during OpCon installation (it can also be a CA certificate), while the Continuous Task Controller Executor Rest-API is a CA supplied certificate.  
The Secret values can be created by running the **install-certificates.ps1** or **install-certificates.sh** script.

```
Usage
cd .\install
.\install-certificates.ps1 `
   -ClientCaCertificate "C:\install\pki\ca.crt" `
	-ServerCertificate "C:\install\pki\public.crt" `
	-ServerKey "C:\install\pki\public.key" `
	-ServingCertificate "C:\install\pki\serving.crt" `
	-ServingKey "C:\install\pki\serving.key" `
	-UpstreamClientCertificate "C:\install\pki\upstream.crt" `
	-UpstreamClientKey "C:\install\pki\upstream.key"

cd .\install
.\install-certificates.ps1 `
   -OpConCaCertificate "C:\install\opcon\ca.crt" 

Usage
chmod +x install-certificates.sh
./install-certificates.sh --client-ca /tmp/pki/ca.crt \
  --server-cert /tmp/pki/public.crt --server-key /tmp/pki/public.key \
  --serving-cert /tmp/pki/serving.crt --serving-key /tmp/pki/serving.key \
  --upstream-cert /tmp/pki/upstream.crt --upstream-key /tmp/pki/upstream.key

chmod +x install-certificates.sh
./install-certificates.sh --opcon-ca /tmp/pki/opcon-ca.crt

```

| Secret | Contents | Purpose |
|---|---|---|
| continuoustask-client-ca | ca.crt | CA that signs the host system's client certificates |
| continuoustask-controller-tls | tls.crt, tls.key | Server certificate for the public ingress hostname |
| continuoustask-serving | tls.crt, tls.key, ca.crt | Certificate the Executor presents in-cluster, plus the CA its callers must chain to |
| continuoustask-upstream-client | tls.crt, tls.key, ca.crt | Client certificate the Ingress presents to the Executor, plus the CA that verifies the Executor's serving certificate |
| continuoustask-opcon-ca | ca.crt | CA the controller trusts for outbound HTTPS to the host system |

##### Environment Variables

These are set in the controller Deployment in continuoustask-controller.yaml.

   | Name | Description |
   |---|---|
   | APIPORT | The internal port number used by the Executor Rest-API server. The default value is 8080. This should be exposed for external Rest-API access. 
   | HOSTSYSTEM | Indicates if the Watcher should be connected to a host system. The values are NONE, OPCON and CONTINUUM (CONTINUUM is not yet implemented). The default is NONE |
   | HOSTADDRESS | Required if HOSTSYSTEM is OPCON and is the address of the OpCon System Rest-API (format address:port). Use the host name that appears in the host's certificate, not an IP address. If the name cannot be resolved by the cluster DNS, add a hostAliases entry to the Deployment |
   | SSL_CERT_FILE | Required if HOSTSYSTEM is OPCON and is the CA trusted for outbound HTTPS to the host system. Default /etc/continuoustask/opcon-ca/ca.crt, supplied by the continuoustask-opcon-ca Secret |
   | HOSTINSECURE | Optional, default false. When true the host system's HTTPS certificate is not validated and HOSTTOKEN is sent to any server at HOSTADDRESS. For development only |
   | HOSTTOKEN | Required if HOSTSYSTEM is OPCON and is an OpCon Rest-API application token used when submitting Rest-API requests. It is read from the continuoustask-host-token Secret (see HOSTTOKEN above) |
   | HOSTAGENT | Required if HOSTSYSTEM is OPCON and is the name of the Continuous Task Agent (ACS "Continuous Task" machine) that is associated with the Continuous Task Controller. When creating OpCon tasks, this value is used as the primary machine |
   | HOSTPROPERTYPREFIXES | Optional, default empty. A comma-separated, case-sensitive list of prefixes (for example "CT_,SHARED_"). A Continuous Resource may reference OpCon properties (`[[name]]` tokens in env vars) only if the property name starts with one of these prefixes. With the default no properties are allowed, and a resource that references one is not exported |
   | KUBEQPS | Sustained requests per second the controller may send to the Kubernetes API. Default 50 |
   | KUBEBURST | Requests that may be sent back-to-back after an idle period. Default 100 |
   | TLSMODE | Certificate authentication for the Executor Rest-API. Values are REQUIRE (HTTPS with a trusted client certificate on every endpoint except heartbeat) and DISABLED (plain HTTP, no authentication). The Deployment sets REQUIRE; the controller's own default is DISABLED |
   | TLSCERTPATH | leave as default /etc/continuoustask/tls/tls.crt |
   | TLSKEYPATH | leave as default /etc/continuoustask/tls/tls.key |
   | TLSCLIENTCAPATH | leave as default /etc/continuoustask/tls/ca.crt |
   | TLSCLIENTSUBJECTS | Optional comma-separated allow-list of client certificate identities. When empty, any certificate issued by the client CA is accepted |

The supplied yaml file contains values from a development environment that must be replaced before use: HOSTSYSTEM, HOSTADDRESS, HOSTAGENT and HOSTPROPERTYPREFIXES, the hostAliases entry (host name to IP address), TLSCLIENTSUBJECTS and the Ingress host name. The Ingress and its network policy expect the nginx ingress controller, installed in the **ingress-nginx** namespace.


### Deployment

1. The first item to deploy within the Kubernetes cluster is the **continuous-system** namespace. 
   All other actions are dependent on the namespace being available.
   The namespace is set by submitting the 00-namespace.yaml file to the Kubernetes cluster.

   ```
   kubectl apply -f ./00-namespace.yaml

   ```
2. The next item to deploy is the secrets and certificates (see Secrets and Certificates section)
3. Deploy the Continuous Task Controller
   This performs the following functions
   - creates the Continuous Custom Resource Definition
   - sets the service account
   - defines the cluster roles
   - defines the continuoustask-author cluster role for people and pipelines that author Continuous tasks (it is not bound to anyone; bind it only to subjects trusted to read the OpCon properties allowed by HOSTPROPERTYPREFIXES)
   - performs the cluster role binding
   - performs the Continuous Task Controller deployment
   - performs the Service binding
   - sets Ingress 


   ```
   kubectl apply -f ./continuoustask-controller.yaml

   ```

   ### Deployment yamml file

  ```
# Single-file install of the ContinuousTask controller: the CRD plus every manifest in deploy/.
# Generated from definitions/crd-continuoustaskV1.yaml and deploy/*.yaml; regenerate rather than edit.
#
#   kubectl apply -f install/continuoustask-controller.yaml
#
# With TLSMODE=REQUIRE the pod will not start until the Secrets from deploy/mtls-secrets.ps1 exist.
# HOSTSYSTEM=OPCON also needs the continuoustask-host-token Secret (key HOSTTOKEN); the pod starts
# without it, but the host system rejects the empty token:
#   kubectl create secret generic continuoustask-host-token -n continuous-system --from-literal=HOSTTOKEN=<token>
---
# Source: deploy/00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: continuous-system
  labels:
    app.kubernetes.io/name: continuoustask-controller
---
# Source: definitions/crd-continuoustaskV1.yaml
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
---
# Source: deploy/10-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: continuoustask-controller
  namespace: continuous-system
  labels:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: executor

---
# Source: deploy/20-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: continuoustask-controller
  labels:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: executor
rules:
  # Watch ContinuousTask resources cluster-wide.
  - apiGroups: ["continuous.com"]
    resources: ["continuoustasks"]
    verbs: ["get", "list", "watch"]
  # Status writes are not performed in v1; granted for the deferred status-subresource enhancement.
  - apiGroups: ["continuous.com"]
    resources: ["continuoustasks/status"]
    verbs: ["get", "update", "patch"]
  # Build/manage Jobs from ContinuousTasks. "patch" is what tasksuspend sets spec.suspend with.
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create", "get", "list", "patch", "delete"]
  # Read the Job status subresource for taskstatus (distinct RBAC resource from jobs).
  - apiGroups: ["batch"]
    resources: ["jobs/status"]
    verbs: ["get"]
  # Enumerate a Job's pods for log aggregation.
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  # Read pod logs (separate resource from pods).
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]

---
# Source: deploy/25-continuoustask-author-clusterrole.yaml
# Permission to write ContinuousTask resources, for the people and pipelines that author tasks.
#
# Creating a ContinuousTask is more privileged than it looks: with HOSTSYSTEM=OPCON its env vars are
# copied into an OpCon job, and OpCon expands any [[name]] token in them when the job runs, so the
# task's container sees that property's value. HOSTPROPERTYPREFIXES (40-deployment.yaml) limits which
# properties can be named, but whoever holds this role can read every property under those prefixes.
# Bind it only to subjects trusted at that level.
#
# Deliberately NOT labelled rbac.authorization.k8s.io/aggregate-to-edit or aggregate-to-admin: that
# would hand it to every namespace editor and admin. Applying this file binds it to nobody.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: continuoustask-author
  labels:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: rbac
rules:
  - apiGroups: ["continuous.com"]
    resources: ["continuoustasks"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Grant it per namespace with a RoleBinding, which confines the subject to that namespace's tasks.
# A ClusterRoleBinding would let the subject author tasks in every namespace.
#
# apiVersion: rbac.authorization.k8s.io/v1
# kind: RoleBinding
# metadata:
#   name: continuoustask-author
#   namespace: <task-namespace>
# roleRef:
#   apiGroup: rbac.authorization.k8s.io
#   kind: ClusterRole
#   name: continuoustask-author
# subjects:
#   - kind: Group
#     name: <trusted-task-authors>
#     apiGroup: rbac.authorization.k8s.io

---
# Source: deploy/30-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: continuoustask-controller
  labels:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: executor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: continuoustask-controller
subjects:
  - kind: ServiceAccount
    name: continuoustask-controller
    namespace: continuous-system

---
# Source: deploy/40-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: continuoustask-controller
  namespace: continuous-system
  labels:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: executor
spec:
  # Exactly one controller per cluster; Recreate ensures a single instance across restarts.
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: continuoustask-controller
      app.kubernetes.io/component: executor
  template:
    metadata:
      labels:
        app.kubernetes.io/name: continuoustask-controller
        app.kubernetes.io/component: executor
    spec:
      serviceAccountName: continuoustask-controller
      # HTTPS to the host system validates its certificate against the name in HOSTADDRESS, so an IP
      # address fails with RemoteCertificateNameMismatch. When the host's certificate names a host the
      # cluster DNS cannot resolve (a Windows machine name, say), map that name to its IP here and put
      # the name, not the IP, in HOSTADDRESS. Check the name with:
      #   openssl s_client -connect <ip>:<port> </dev/null | openssl x509 -noout -subject -ext subjectAltName
      hostAliases:
        - ip: "192.0.2.10"
          hostnames:
            - "opcon-server"
      securityContext:
        runAsNonRoot: true
      containers:
        - name: controller
          # Built and pushed by ci.yml; a v* git tag produces the matching image tag.
          # Bump this tag on every code change: with IfNotPresent, re-pushing the same tag leaves the
          # node's cached image in place and the rollout silently keeps running the old build.
          image: smaengineering.azurecr.io/continuoustask-controller:2.0.0.96826
          imagePullPolicy: IfNotPresent
          ports:
            - name: http-api
              containerPort: 8080
          env:
            - name: APIPORT
              value: "8080"
            - name: HOSTSYSTEM
              value: "NONE"
            - name: HOSTADDRESS
              value: ""
            # Extra CA trusted for outbound HTTPS, for a host system whose certificate is self-signed
            # or issued by an internal CA (RemoteCertificateChainErrors otherwise). .NET still loads
            # the image's /etc/ssl/certs directory, so public CAs stay trusted. The file is absent,
            # and ignored, until the continuoustask-opcon-ca Secret exists (mtls-secrets.ps1
            # -OpConCaCertificate). Restart the pod after creating or rotating it.
            - name: SSL_CERT_FILE
              value: "/etc/continuoustask/opcon-ca/ca.crt"
            # DEVELOPMENT ONLY. "true" skips validation of the host system's HTTPS certificate
            # entirely, so HOSTTOKEN goes to whatever server answers at HOSTADDRESS. Prefer
            # SSL_CERT_FILE / hostAliases above; the controller logs a warning at startup while it is on.
            - name: HOSTINSECURE
              value: "false"
            # Credential for the host system. Held in the continuoustask-host-token Secret, created out
            # of band (install/install-hosttoken.ps1). Optional so HOSTSYSTEM=NONE needs no Secret; with
            # the Secret absent the token is empty, so under HOSTSYSTEM=OPCON the host system rejects
            # calls instead of the pod failing to start. Restart the pod after creating or rotating it.
            - name: HOSTTOKEN
              valueFrom:
                secretKeyRef:
                  name: continuoustask-host-token
                  key: HOSTTOKEN
                  optional: true
            # Host-system agent that runs the tasks: for OPCON, the ACS "Continuous Task" machine every
            # OpCon job is assigned to. Unused while HOSTSYSTEM=NONE.
            - name: HOSTAGENT
              value: ""
            # Comma-separated, case-sensitive OpCon property-name prefixes (e.g. "CT_,SHARED_") that a
            # ContinuousTask's env vars may reference as [[name]] tokens. OpCon expands those tokens at
            # launch into the task's environment, so this bounds which properties task authors can read.
            # Empty allows none: a task referencing any property is logged and not exported. "$" would
            # allow OpCon's system properties. Who may author tasks: 25-continuoustask-author-clusterrole.yaml.
            - name: HOSTPROPERTYPREFIXES
              value: ""
            # Client-side ceiling on requests to the API server. Requests over the limit wait; they
            # are never dropped. Raise only if a large cluster makes the controller visibly slow.
            - name: KUBEQPS
              value: "50"
            - name: KUBEBURST
              value: "100"
            # Certificate authentication on the Executor API. REQUIRE means the listener serves HTTPS
            # and every endpoint except the heartbeat demands a client certificate that chains to
            # TLSCLIENTCAPATH. The pod will not start if any of the three files below is missing --
            # deliberately, so a broken mount cannot silently downgrade the API to plaintext.
            #
            # Set to DISABLED for a local cluster with no certificates to hand; the API then behaves
            # exactly as it did before this was added (plain HTTP, no authentication), and the probes
            # below must go back to scheme: HTTP.
            - name: TLSMODE
              value: "REQUIRE"
            - name: TLSCERTPATH
              value: "/etc/continuoustask/tls/tls.crt"
            - name: TLSKEYPATH
              value: "/etc/continuoustask/tls/tls.key"
            - name: TLSCLIENTCAPATH
              value: "/etc/continuoustask/tls/ca.crt"
            # Comma-separated CNs or SHA-1 thumbprints; leaving it empty would accept any certificate
            # the CA has issued. The first entry must be the CN of the client certificate in the
            # continuoustask-upstream-client Secret, or requests through the Ingress are refused.
            #
            # Every additional entry is a caller allowed to reach the Executor *directly* on the
            # Service, skipping the Ingress and therefore its auth-tls check. That is a deliberate
            # choice per caller, not a default: with one CA issuing everything, a CN listed here can
            # present its certificate straight at the pod.
            - name: TLSCLIENTSUBJECTS
              value: "ingress-nginx,opcon.example.com"
          volumeMounts:
            - name: tls
              mountPath: /etc/continuoustask/tls
              readOnly: true
            - name: opcon-ca
              mountPath: /etc/continuoustask/opcon-ca
              readOnly: true
          # Probes are HTTPS because the listener is. The kubelet cannot present a client certificate,
          # which is why /heartbeat is the one endpoint exempt from authentication; it does not verify
          # the server certificate either, so the serving cert's SANs do not have to cover the pod IP.
          readinessProbe:
            httpGet:
              path: /continuoustc/api/heartbeat
              port: http-api
              scheme: HTTPS
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /continuoustc/api/heartbeat
              port: http-api
              scheme: HTTPS
            initialDelaySeconds: 10
            periodSeconds: 15
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
            capabilities:
              drop:
                - ALL
            seccompProfile:
              type: RuntimeDefault
      volumes:
        # Serving certificate plus the CA that client certificates must chain to. Created out of band
        # by mtls-secrets.ps1, so the Deployment can be applied from this repo without shipping keys.
        #
        # 0444 rather than 0400 on purpose: Secret volume files are owned by root, the container runs
        # as the image's non-root user, and 0400 would leave the key unreadable by the process that
        # needs it. Narrowing further would mean pinning fsGroup to the base image's APP_UID group,
        # which silently breaks if that image changes. Nothing but this controller runs in the pod.
        - name: tls
          secret:
            secretName: continuoustask-serving
            defaultMode: 0444
        # CA for the host system's HTTPS certificate (see SSL_CERT_FILE). Optional, unlike tls above:
        # a host system with a publicly trusted certificate needs nothing here, and HOSTSYSTEM=NONE
        # makes no outbound calls at all.
        - name: opcon-ca
          secret:
            secretName: continuoustask-opcon-ca
            optional: true
            defaultMode: 0444

---
# Source: deploy/50-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: continuoustask-controller
  namespace: continuous-system
  labels:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: executor
spec:
  # ClusterIP only. External callers come through the mTLS Ingress (70-ingress.yaml); do not change
  # this to NodePort or LoadBalancer, which would publish the API with the certificate check bypassed.
  # The Executor itself is still unauthenticated on this port, so 60-networkpolicy.yaml limits who may
  # reach it to the ingress controller.
  type: ClusterIP
  selector:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: executor
  ports:
    # targetPort references the named container port, so it follows APIPORT via the pod spec.
    - name: http-api
      port: 8080
      targetPort: http-api
      protocol: TCP

---
# Source: deploy/60-networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: continuoustask-controller-api
  namespace: continuous-system
  labels:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: executor
spec:
  # This policy is what turns ingress-terminated mTLS into an actual control. The ingress controller
  # verifies client certificates, but the Executor itself is still unauthenticated: without this
  # policy any pod in the cluster could reach the ClusterIP Service (or the pod IP) directly and skip
  # the certificate check entirely.
  #
  # Requires a CNI that enforces NetworkPolicy (Calico, Cilium, Antrea, ...). On a cluster whose CNI
  # ignores NetworkPolicy this object applies cleanly and silently does nothing — verify enforcement
  # before relying on it (see the check in mtls-secrets.ps1 output).
  podSelector:
    matchLabels:
      app.kubernetes.io/name: continuoustask-controller
      app.kubernetes.io/component: executor
  # Ingress only. The controller's own outbound traffic (API server watch, Job CRUD, future host-system
  # calls) is deliberately left unrestricted; adding Egress here would need the API server endpoint
  # and cluster DNS allowed explicitly.
  policyTypes:
    - Ingress
  ingress:
    # The ingress controller is the only permitted client. Adjust the namespace if ingress-nginx is
    # installed somewhere other than `ingress-nginx` — the kubernetes.io/metadata.name label is set
    # automatically by the API server, so the namespace needs no extra labelling.
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        # Named port, so this follows APIPORT through the pod spec exactly as the Service does.
        - port: http-api
          protocol: TCP
    # Kubelet probes and `kubectl port-forward` arrive from the node, not from a pod, and some CNIs
    # (Calico with a default-deny host policy, Cilium without `host` in the allow set) filter that
    # traffic under this policy. If the readiness/liveness probes in 40-deployment.yaml start failing
    # or deploy.ps1 -SmokeTest can no longer reach the heartbeat, uncomment this and set the node CIDR
    # for the cluster. Note this re-opens the API to anything that can source traffic from a node.
    # - from:
    #     - ipBlock:
    #         cidr: 10.0.0.0/16   # replace with the cluster's node CIDR
    #   ports:
    #     - port: http-api
    #       protocol: TCP

---
# Source: deploy/70-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: continuoustask-controller
  namespace: continuous-system
  labels:
    app.kubernetes.io/name: continuoustask-controller
    app.kubernetes.io/component: executor
  annotations:
    # --- Mutual TLS -------------------------------------------------------------------------------
    # The ingress controller demands a client certificate and verifies it against the CA in
    # auth-tls-secret. Requests with no certificate, an expired one, or one from another CA are
    # rejected at the edge with 400 and never reach the Executor.
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    # Must live in the same namespace as this Ingress: cross-namespace secret references are refused
    # unless the ingress controller runs with --allow-cross-namespace-resources.
    # Created by mtls-secrets.ps1, not by `kubectl apply -f deploy/` — it holds your CA, not ours.
    nginx.ingress.kubernetes.io/auth-tls-secret: "continuous-system/continuoustask-client-ca"
    # 1 = client certificates must be signed directly by that CA. Raise only for an intermediate chain.
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
    # The Executor does not inspect the certificate, so it is not forwarded. Set to "true" if per-caller
    # authorization or audit logging is added later — the app would then read ssl-client-subject-dn /
    # ssl-client-cert, and must reject those headers on any path that does not come via this Ingress.
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "false"
    # Restrict *which* certificates from that CA may call, instead of trusting every certificate it
    # ever issued. Uncomment once the caller's CN is known; the value is a regex matched against the
    # client cert subject.
    # nginx.ingress.kubernetes.io/auth-tls-match-cn: "CN=(opcon|continuum)\\.example\\.com"
    # --- Transport --------------------------------------------------------------------------------
    # No plaintext path in: HTTP is redirected at the edge, and the hop to the Executor is HTTPS with
    # its own mutual authentication (TLSMODE=REQUIRE in 40-deployment.yaml). The caller's certificate
    # is verified here and a *separate* identity is presented upstream, so neither layer is a single
    # point of bypass: a caller that reaches the Service directly still has no accepted certificate,
    # and the ingress cannot be impersonated without the key below.
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # Client certificate the ingress presents to the Executor. Its CN must appear in the controller's
    # TLSCLIENTSUBJECTS. The same Secret's ca.crt verifies the Executor's serving certificate, so this
    # hop is authenticated in both directions.
    nginx.ingress.kubernetes.io/proxy-ssl-secret: "continuous-system/continuoustask-upstream-client"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "on"
    nginx.ingress.kubernetes.io/proxy-ssl-verify-depth: "1"
    # Verified against the Executor certificate's SANs, and sent as SNI. The serving certificate must
    # therefore carry this name — see mtls-secrets.ps1.
    nginx.ingress.kubernetes.io/proxy-ssl-name: "continuoustask-controller.continuous-system.svc"
    nginx.ingress.kubernetes.io/proxy-ssl-server-name: "on"
spec:
  ingressClassName: nginx
  tls:
    # Server certificate for the public hostname — separate from the client CA above. Also created by
    # mtls-secrets.ps1.
    - hosts:
        # REPLACE controller.example.com with the hostname the host system calls, here and in the rule
        # below, and issue the serving certificate (mtls-secrets.ps1) for that name. It is a placeholder:
        # left as is, requests for the real hostname match no rule and get a 404 from the ingress.
        # Use lowercase: the API server rejects uppercase hosts, and nginx matches the Host header
        # case-insensitively anyway.
        - controller.example.com
      secretName: continuoustask-controller-tls
  rules:
    - host: controller.example.com
      http:
        paths:
          # Endpoints are listed individually rather than as one /continuoustc/api prefix so that
          # /continuoustc/api/heartbeat is *not* published: probes reach it directly from the kubelet,
          # and it reports host-system config and cache occupancy that external callers need not see.
          # The backend port is the Service's named port, so it follows APIPORT like everything else.
          - path: /continuoustc/api/taskstart
            pathType: Prefix
            backend:
              service:
                name: continuoustask-controller
                port:
                  name: http-api
          - path: /continuoustc/api/taskstatus
            pathType: Prefix
            backend:
              service:
                name: continuoustask-controller
                port:
                  name: http-api
          - path: /continuoustc/api/tasklogs
            pathType: Prefix
            backend:
              service:
                name: continuoustask-controller
                port:
                  name: http-api
          - path: /continuoustc/api/tasksuspend
            pathType: Prefix
            backend:
              service:
                name: continuoustask-controller
                port:
                  name: http-api
          - path: /continuoustc/api/taskcleanup
            pathType: Prefix
            backend:
              service:
                name: continuoustask-controller
                port:
                  name: http-api
  ```
  