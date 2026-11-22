---
slug: '/'
sidebar_label: 'OpCon Agent Installation'
---

# Installation

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