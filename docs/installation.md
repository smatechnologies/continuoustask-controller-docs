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
   | APIPORT | The internal port number used by the Executor Rest-API server. The default value is 8080. The local service (55-service-local.yaml section) maps it to 18080 |
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
   - performs the local service binding
   - sets network policies
   - sets Ingress 


   ```
   kubectl apply -f ./continuoustask-controller.yaml

   ```

## OpCon Continuous Task

The OpCon Continuous Task software can be downloaded from the SMA FTP Site (see Installation below).

### ACS installation and upgrade

**Theme:** Build
**Who is it for?** System Administrator

### What is it?

The OpCon Continuous Task integrates an OpCon system with the Continuous Task Controller (custom Kubernetes Job Controller). Installing the connector deploys the required plugin files to the OpCon environment, making the connector available for agent and job configuration in Solution Manager.

- Use this procedure when setting up the connector for the first time on a new OpCon environment
- Use the upgrade procedure when a new connector version is available and you need to replace the existing plugin files

#### Installation

To install the Continuous Task, complete the following steps:

1. Download the `ACSContinuousTask.zip` file from the SMA FTP Site. The file is located at one of the following paths:
   - `/OpCon Releases/Integrations/ACSContinuousTask/`
2. Select the required version.
3. Unzip the `ACSContinuousTask.zip` file.
4. Complete the steps for your deployment type:

   | Deployment type | Steps |
   |---|---|
   | On-prem | Copy the `ACSContinuousTask` directory to the `\\SAM\\plugins` directory. Restart the **SMA OpCon Services Manager** and **SMA OpCon RestAPI** service. |
   | Cloud | Copy the `ACSContinuousTask` directory to the `\\Relay\\plugins` directory. Restart the **SMA OpCon Relay** service. |

The connector is installed and available for configuration in Solution Manager.

#### Configuring the agent

In Solution Manager, define the Continuous Task agent with the following settings:

| Setting | Description |
|---|---|
| API Address (required) | Address of the Continuous Task Controller Rest-API |
| API Port (required) | Port of the Continuous Task Controller Rest-API |
| Retain Log Files | Days to retain log files. Default 30 |
| Client Certificate | Full path to a PFX client certificate to present to the Controller. Leave empty to connect without one. Requires HTTPS |
| Certificate Password | Password protecting the PFX file. Use an encrypted global property to protect it |
| Validate Server Certificate | Rejects the Controller certificate if it is untrusted, expired or does not match the API Address. Default off; leave it off for a self-signed Controller certificate |

When the Controller runs with TLSMODE=REQUIRE (the deployment default), the agent must present a client certificate issued by the CA held in the **continuoustask-client-ca** Secret, and the certificate identity must be allowed by the Controller's TLSCLIENTSUBJECTS setting (see Environment Variables).

#### Upgrade

To upgrade the OpCon Continuous Task, complete the following steps:

1. Download the `ACSContinuousTask.zip` file from the SMA FTP Site. The file is located at one of the following paths:
   - `/OpCon Releases/Integrations/ACSContinuousTask/`
2. Unzip the `ACSContinuousTask.zip` file.
3. Complete the steps for your deployment type:

   | Deployment type | Steps |
   |---|---|
   | On-prem | Stop the **SMA OpCon Services Manager** and **SMA OpCon RestAPI** service. Copy the `ACSContinuousTask` directory to the `\\SAM\\plugins` directory. Restart the **SMA OpCon Services Manager** and **SMA OpCon RestAPI** service. |
   | Cloud | Copy the `ACSContinuousTask` directory to the `\\Relay\\plugins` directory. Restart the **SMA OpCon Relay** service. |

The connector is upgraded to the new version.