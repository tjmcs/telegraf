# Basic deployment scenarios

There are a two basic deployment scenarios that are supported by this playbook. In the first scenario (shown below) we'll walk through the deployment of Telegraf agents to one or more nodes in an AWS or OSP cloud environment using dynamic inventory information gathered from that environment. In the second scenario, we will show how that same deployment could be performed to a set of one or more nodes in a non-cloud environment using a static inventory file. In both of these scenarios there is an assumption that the set of target nodes already exists. Unlike the other application playbooks that we have developed, the [provision-telegraf.yml](../provision-telegraf.yml) playbook in this repository does not support the creation of the nodes that will be targeted by a given playbook run during that playbook run. The target nodes must already exist to successfully deploy Telegraf agents to those nodes.

## Scenario #1: deployment to an existing set of AWS or OpenStack nodes
In this section we will describe the deployment of Telegraf agents to a set of VMs that already exist in a targeted AWS or OpenStack environment. This makes the configuration file that we will use much simpler, since the hard work of actually building and configuring the VMs (placing them on the right network, attaching the necessary data volume, etc.) has already been done. As such, the playbook run is be reduced to the following two tasks:

* **building the application host groups**: the targeted application and `kafka` host groups will be built dynamically from the inventory available in the targeted AWS or OpenStack environment, based on the tags that are passed into the playbook run
* **deploying Telegraf agents to the target nodes**: once the matching nodes have been identified, the playbook deploys Telegraf agents to those nodes and configures them to report the metrics that they gather back to the associated Kafka cluster

To accomplish this, the we have to:

* construct a configuration file that can be used to control the deployment based on the `Cloud`, `Tenant`, `Project`, `Dataflow`, `Domain`, `Application`, and `Cluster` tags of the target nodes for that deployment
* run a playbook command (the format of this command will vary slightly depending on whether the target environment is an AWS or OpenStack environment) to deploy Telegraf agents to the target nodes and configure them appropriately.

In terms of what the command looks like, lets assume for this example that we're targeting the same nodes that make up the Kafka cluster for deployment of a set of Telegraf agents and that those nodes are already running in an AWS with the following VM tags:

* **Cloud**: aws
* **Tenant**: datanexus
* **Project**: demo
* **Dataflow**: none
* **Domain**: production
* **Application**: kafka
* **Cluster**: a

Furthermore, let's assume that the VMs in question are running in the `us-west-2` EC2 region in a VPC and with a network configuration that matches the values defined defined in the [vars/telegraf.yml](../vars/telegraf.yml) variables file. To target these nodes, we would setup a custom configuration file that might look something like this:

```yaml
$ cat config-aws-custom.yml
---
# target application for the deployment
application: kafka
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
cluster: a
dataflow: pipeline
# network configuration used when configuring the agents
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
```

If it was necessary to override any of the default values from the [vars/telegraf.yml](../vars/telegraf.yml) file in your own playbook run (to target machines on a different set of subnets, in a different VPC, and/or in a different region, for example) then those parameters should be added to a custom configuration file shown above.

Assuming that we have created our VMs in the `us-west-2` EC2 region in the proper VPC, placed them on the proper subnets within that VPC, and tagged them appropriately, then the command needed to deploy Telegraf agents to those nodes would be quite simple:

```bash
$ AWS_PROFILE=datanexus_west provision-telegraf.yml -e '{ config_file: "./config-aws.yml" }'
```

This command would read in the tags from the custom configuration file (shown above), find the matching VMs in the `us-west-2` EC2 region, and deploy a set of Telegraf agents to those nodes (configuring those agents to report the metrics that they collect to the associated Kafka cluster).

The playbook command used to deploy Telegraf agents to the same set of Kafka nodes in an OpenStack environment would look quite similar:

```bash
$ provision-telegraf.yml -e "{ \
        config_file: config-osp.yml \
    }"        
```

where the configuration file we are passing in would look nearly the same as the default configuration file used in the AWS deployment:

```yaml
$ cat config-osp.yml
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
user: cloud-user
# path to directory containing the private keys needed to access
# the target nodes
private_key_path: '/tmp/keys'
```

As you can see, this file doesn't differ significantly from the default configuration file that was shown previously, other than:

* changing the value of the `cloud` parameter to `osp` from `aws`
* modifying the default value for the `user` parameter from `centos` to `cloud-user`; the value shown here is meant to be an example, and the actual value for this parameter may differ in your specific OpenStack deployment (check with your OpenStack administrator to see what value you should be using)
* setting the `private_key_path` to a directory that contains the keys needed to access instances in your OpenStack environment; by default this parameter is set to the playbook directory, so if you keep the keys needed to SSH into the target instances in your OpenStack environment in the top-level directory of the repository this parameter is not necessary

As was the case with the AWS deployment example shown previously, we are assuming here that a set of nodes already exists in the targeted OpenStack environment and that those nodes have been tagged appropriately. The matching set of nodes will then be used to construct the list of nodes will be targeted by the playbook run.

## Scenario #2: using static inventory file to control the playbook run
If you are using this playbook to deploy Telegraf agents to one or more target nodes in a non-cloud environment (i.e. an environment other than an AWS or OpenStack environment), then a static inventory file must be passed into the playbook to control which nodes will be targeted by the playbook run and to define how Ansible should connect to those nodes. In this example, let's assume that we are targeting three (pre-existing) Zookeeper nodes in our playbook run and that those nodes are already up and running with the IP addresses 192.168.34.18, 192.168.34.19, and 192.168.34.20. In that case, the static inventory file that we will be using might look something like this:

```bash
$ cat combined-inventory
# example combined inventory file for clustered deployment

192.168.34.8 ansible_ssh_host=192.168.34.8 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.9 ansible_ssh_host=192.168.34.9 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.10 ansible_ssh_host=192.168.34.10 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'

192.168.34.18 ansible_ssh_host=192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host=192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host=192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

[kafka]
192.168.34.8
192.168.34.9
192.168.34.10

[zookeeper]
192.168.34.18
192.168.34.19
192.168.34.20

```

To deploy Telegraf agents to the three Zookeeper nodes in our static inventory file, we'd run a command that looks something like this:

```bash
$ provision-telegraf.yml -i combined-inventory -e "{ \
      config_file: /dev/null,  application: zookeeper
    }"
```

Alternatively, rather than passing in the value for the `application` parameter and any other parameters needed during our playbook run as extra variables on the command-line, we can make use of the *configuration file* support that is built into our application deployment playbooks and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
---
application: zookeeper
```

we could then pass that that *configuration file* into the playbook run as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-config.yml`, the resulting command would look something like this:

```bash
$ provision-telegraf.yml -i combined-inventory -e "{ \
      config_file: 'test-config.yml' \
    }"
```

Once the playbook run is complete, we can simply SSH into one of the nodes in our Kafka cluster (eg. 192.168.34.9) and use the `kafka-console-consumer` to check and make sure tha the metrics collected by our Telegraf agents are being written to our Kafka cluster by those agents.
