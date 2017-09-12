# Dynamic vs. static inventory
The first decision to make when using the playbook in this repository to deploy Telegraf agents to a set of target nodes is whether you would like to manage the inventory for your deployments dynamically or statically. Since the static inventory use case is the simplest, we'll start our discussion there. Then we'll move on to a discussion of how to control the deployment of Telegraf agents in both AWS and OpenStack environments, where the list of nodes that should be created and targeted by a given `ansible-playbook` run is constructed dynamically from the input configuration file.

## Managing deployments with static inventory files
Let's assume that we are planning to deploy Telegraf agents to three Zookeeper nodes using the playbook in this repository and we are planning on using a [static inventory file](https://docs.ansible.com/ansible/intro_inventory.html) to control that deployment. In this example (and others like it in this document) we'll assume that we're going to construct a single static inventory file (an INI-like formatted file containing the list of hosts that are targeted by any given playbook run), and then pass that file into the `ansible-playbook` command using the `-i, --inventory-file` command-line flag.

In our discussion of the various deployment scenarios supported by this playbook, we show that deployments of Telegraf agents to nodes actually require an associated (assumed to be external) Kafka cluster that those agents will report the logs/metrics that they gather to. For purposes of this discussion, we will focus only on the inventory information associated with the nodes we are deploying our Telegraf agents to, not on the inventory information associated with the Kafka cluster. So, returning to our example, the inventory file associated with our three target nodes might look something like this:

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

As you can see, our static inventory file consists of a list of the hosts involved in our Telegraf deployment followed by two host groups (the `zookeeper` and `kafka` host groups in this example). For each host in this file, we provide a list of the parameters that Ansible will need to connect to that host (INI-file style) as a set of `name=value` pairs. In this example, we've defined the following values for each of the entries in our static inventory file:

* **`ansible_ssh_host`**: the hostname/address that Ansible should use to connect to that host; if not specified, the same hostname/address listed at the start of the line for that entry for that host will be used (in this example we are using IP addresses). This parameter can be important when there are multiple network interfaces on each host and only one of them (an admin network, for example) allows for SSH access
* **`ansible_ssh_port`**: the port that Ansible should use when connecting to the host via SSH; if not specified, Ansible will attempt to connect using the default SSH port (port 22)
* **`ansible_ssh_user`**: the username that Ansible should use when connecting to the host via SSH; if not specified, then the value of the global `ansible_user` variable will be used (this variable can be set as an extra variable in the `ansible-playbook` command if the same username is used for all hosts targeted by the playbook run)
* **`ansible_ssh_private_key_file`**: the private key file that should be used when connecting to the host via SSH; if not specified, then the value of the global `ansible_ssh_private_key_file` variable will be used instead (this variable can be set as an extra variable in the `ansible-playbook` command if the same private key is used for all hosts targeted by the playbook run)

With this static inventory file built, it's a relatively simple matter to provision Telegraf agents to those nodes using a single `ansible-playbook` command. Examples of these commands are shown [here](Deployment-Scenarios.md).

## Managing deployments using dynamic inventory scripts
For both AWS and OpenStack environments, the playbook in this repository supports the use of a [dynamic inventory](https://docs.ansible.com/ansible/intro_dynamic_inventory.html) to control deployments to those environments. This is accomplished by making use of the [build-app-host-groups](../roles/build-app-host-groups) role, which builds the host groups that are needed for the playbook run by filtering the hosts in the AWS or OpenStack environment, based on the tags that are assigned them and the tags that were included in the `ansible-playbook` command. This provides an attractive alternative to building static inventory files to control the deployment process in these environments, since the meta-data needed to determine which nodes should be targeted by a given playbook run is readily available in the framework itself.

The process of using the [build-app-host-groups](../roles/build-app-host-groups) role to control a playbook run starts out by identifying values for the tags of the VMs that are going to be the targeted by a particular deployment. Specifically, this role constructs the host groups used in the plaubook run based on the following tags:

* **tenant**: the tenant that is using the nodes being targeted. As an example, you might use a value of `labs` for this tag for nodes that the Labs department in your organization will be using
* **project**: this tag will be given the value of the project that will be using the nodes that are geing targeted; for example we might use the value `projectx` for this tag for clusters that will be used for ProjectX
* **dataflow**: this tag is optional, and is used to identify the various clusters/ensembles (Cassandra, Zookeeper, Kafka, Solr, etc.) that make up a given data flow. To link these applications together, it is necessary to define a unique `dataflow` value for each of the ensembles/clusters that are are being linked together. If a value for this tag is not defined, then a default value of `none` will be used when searching for matching VMs in an AWS or OpenStack environment.
* **domain**: this tag is used to separate out the various domains that might be defined by the project team to control their deployments. For example, there might be separate groups of machines built for `development`, `test`, `preprod`, and `production`. Each of these environments would be tagged appropriately based on their intended use.
* **cluster**: this tag is optional, and is only necessary if the intent is to deploy more than one cluster or ensemble for the same tenant, project, data flow, domain, and application. In that case, it is necessary to define a unique `cluster` value for each of the clusters or ensembles that are are being targeted using the [provision-telegraf.yml](../provision-telegraf.yml) playbook. If a value for this tag is not defined, then a default value of `a` will be used when searching for matching VMs in an AWS or OpenStack environment.
* **application**: used to identify the application that is being targeted for deployment; this value defaults to the value of `telegraf` defined in the [vars/telegraf.yml](../vars/telegraf.yml) variables file, but this value should be modified to reflect the `application` tag associated with the nodes being targeted by a given playbook run
* **role**: the role is used in some of our playbooks to separate out the nodes that have a different role in that cluster from the other roles in the cluster; for example in a Cassandra cluster some of the nodes are seed nodes, so they are tagged with a `Role` tag of `seed`; a default value of `none` is used if this value is not specified during a given playbook run

These input values are used to search for a matching set of nodes in the target environment. If a matching set of nodes is found (based on the corresponding `Tenant`, `Project`, `Dataflow`, `Domain`, `Cluster`, `Application`, and `Role` tags), then those machines will be assumed to be the target of the [provision-telegraf.yml](../provision-telegraf.yml) playbook run.

It should be noted here that this playbook assumes that an associated Kafka cluster has already been deployed that the Telegraf agents we are deploying can report the metrics/logs that they will gather to. In the dynamic inventory use case, the nodes that make up this Kafka cluster must be tagged with the same `tenant`, `project`, `dataflow`, and `domain` tags that are used by the nodes we are targeting in a given playbook run. Obviously, these Kafka instances would have been tagged with an `application` tag of `kafka`, but if the remaining tags do not match then the playbook will not be able to find the associated Kafka cluster and the Telegraf agents we are deploying will fail to start.

With these values assigned as part of the `ansible-playbook` command (or in a or in a *custom configuration file*), the [build-app-host-groups](../roles/build-app-host-groups) role that is used by this playbook can correctly identify the nodes in the AWS or OpenStack environment that are targeted by a given playbook run and then use the resulting (dynamically constructed) lists of targeted application (and the nodes that make up the associated Kafka cluster) to control the deployment and configuration of Telgraf agents to those target nodes.
