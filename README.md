# telegraf
Playbooks/Roles used to deploy Telegraf agents

# Installation
To install a Telegraf agent (or agents) using the [provision-telegraf.yml](provision-telegraf.yml) playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone https://github.com/Datanexus/telegraf
```

That command will pull down the repository, which includes the [provision-telegraf.yml](provision-telegraf.yml) playbook and all of its dependencies.

The only other requirements for using the playbook in this repository are a relatively recent (v2.x) release of Ansible. The easiest way to obtain a recent relese if Ansible is via a `pip install`, which requires that Python and pip are both installed locally. We have performed all of our testing using a recent (2.7.x) version of Python (Python 2); your mileage may vary if you attempt to run the playbook or the attached dynamic inventory scripts under a newer (v3.x) release of Python (Python 3).

# Using this role to deploy Telegraf agents
The [provision-telegraf.yml](provision-telegraf.yml) file at the top-level of this repository supports both single-node and the multi-node Telegraf agent deployments. The process of deploying Telegraf agents to these nodes will vary, depending on whether you are managing your inventory dynamically or statically (more on this topic [here](docs/Dynamic-vs-Static-Inventory.md)), whether you are performing a single-node or multi-node deployment, and where you are downloading the packages and dependencies from that are needed to run a Telegraf agent on those nodes.

We discuss the various deployment scenarios supported by this playbook in [this document](docs/Deployment-Scenarios.md). With no changes to the the contents of repository (more on this below), the [provision-telegraf.yml](provision-telegraf.yml) playbook can be used (out of the box, so to speak) to deploy Telegraf agents onto a set of pre-existing Kafka nodes in an AWS environment using a command that looks something like this:

```bash
$ AWS_PROFILE=datanexus_west ./provision-telegraf.yml -e '{ application: kafka }'
```

The `AWS_PROFILE` environment variable value shown above will likely need to change to reflect the profiles configured in your local AWS credentials file.

The command used to deploy Telegraf agents to the same set of pre-existing nodes in an OSP environment is similar, but additional configuration parameters must be passed into the playbook (above and beyond those used for an AWS deployment). The easiest way to do this is to pass in a customized configuration file as an extra variable on the command-line of your Ansible playbook run:

```bash
$ ./provision-telegraf.yml -e '{ config_file: "./config-osp.yml" }'
```

here, the `./config-osp.yml` file is assumed to be a configuration file that you have created in the current working directory that contains all of the additional parameters needed for an OSP deployment. For example, you might create a file that looks something like this:

```yaml
$ cat ./config-osp.yml
---
# target application for the deployment
application: kafka
# cloud type and VM tags
cloud: osp
tenant: datanexus
project: demo
domain: production
application: kafka
cluster: a
dataflow: pipeline
# network configuration used when configuring the agents
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'cloud-user'
# the HTTP proxy-related parameters
http_proxy: http://proxy.datanexus.corp:8080
proxy_certs_file: eg-certs.tar.gz
no_proxy: datanexus.corp
```

As you can see, there are a number of additional parameters needed for an OSP deployment some of the default values from the [vars/telegraf.yml](../vars/telegraf.yml) file and the default configuration file (the [config.yml](../config.yml) file) are overridden by the values defined in this example configuration file (the `cloud`, `tenant`, `project`, `domain`, `internal_subnet`, `external_subnet`, and `user` values). There are also a few variables defined in this configuration file that are not defined in either the [vars/telegraf.yml](../vars/telegraf.yml) file or the default configuration file used when deploying to an AWS environment (the `http_proxy`, `proxy_certs_file`, and `no_proxy` parameters). All of the values shown in this example file are for demonstration purposes only, and the correct values to use for these parameters will change from environment to environment. Check with the your local OSP administrator to determine the correct values to use for these parameters in your OSP environment.

## Controlling the configuration
As was mentioned briefly in the prevision section, this repository includes default values for a number of parameters in the [vars/telegraf.yml](vars/telegraf.yml) file and the default configuration file (the [config.yml](config.yml) file) that make it possible to perform deployments Telegraf agents to pre-existing nodes, out of the box, with few (if any) changes necessary. If you are not happy with the default configuration defined in these two files, there are a number of ways that you can customize the configuration used for your deployment, and which method you use is entirely up to you:

* You can edit the [vars/telegraf.yml](vars/telegraf.yml) file and/or [config.yml](config.yml) file to modify the default values that are defined in those files or define additional configuration parameters
* You can pass in values as extra variables on the command-line of the Ansible playbook run to override those defined in the [vars/telegraf.yml](vars/telegraf.yml) file; values defined in the default configuration file cannot be overridden in this manner since that file is loaded last by the plays in the playbook (so values defined in a configuration file override any values defined elsewhere)
* You can setup a custom *configuration file* on the local filesystem of the Ansible host that contains the values for the parameters you wish to set or customize, then pass the location of that file into your playbook command as an extra variable

We have provided a summary of the configuration parameters that can be set (using any of these three methods) during a playbook run [here](docs/Supported-Config-Params.md). Overall, we have found the last option to be the easiest and most flexible of those three options because:

* It avoids modifying files that are being tracked under version control in the main, [telegraf repository](https://github.com/Datanexus/telegraf) (the first option); making such changes will, more than likely, lead to conflicts at a later date when these files are modified in the main [telegraf repository](https://github.com/Datanexus/telegraf) in a way that is inconsistent with the values that you have set locally (in your clone of this repository).
* It lets you maintain your preferred configuration for any given Telegraf deployment in the form of a configuration file, which you can easily maintain (along with the configuration files used for other deployments you have made) under version control in a separate repository
* It provides a record of the configuration of any given deployment, which is in direct contrast to the second option (where the configuration parameters for any given deployment are passed in on the command-line as extra variables)

That being said, the second option may be useful for some deployment scenarios (a one-off deployment of a local test environment, for example), so it remains a viable option for some users. Overall, we would recommend against trying to maintain your preferred ensemble configuration using the values defined in the [vars/telegraf.yml](vars/telegraf.yml) file or the default configuration file (the [config.yml](config.yml) file).

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions or earlier versions of these distributions (the [provision-telegraf.yml](provision-telegraf.yml)  playbook will not run successfully).
