# Supported configuration parameters
The playbook in the [provision-telegraf.yml](../provision-telegraf.yml) file in this repository pulls in a set of default values for many of the configuration parameters that are needed to deploy Zookeeper from the [vars/telegraf.yml](../vars/telegraf.yml) file and the default configuration file (the [config.yml](../config.yml) file). The parameters defined in these files define a reasonable set of defaults for a fairly generic deployment of Telegraf agents to a set of target nodes, including defaults for the metrics and logs that should be gathered, the topics that those metrics and logs should be pushed to, and whether communications between the Telegraf agents and the associated Kafka cluster should be secured.

While you may never need to change most of these values from their defaults, there are a fairly large number of these parameters, so a brief summary of what each is and how it is used could be helpful. In this section, we summarize all of these options, breaking them out into:

* parameters used to control the Ansible playbook run
* parameters used during the deployment process itself, and
* parameters used to configure the Telegraf agents once those agents have been installed

Each of these sets of parameters are described in their own section, below.

## Parameters used to control the playbook run
The following parameters can be used to control the `ansible-playbook` run itself, defining things like how Ansible should connect to the nodes involved in the playbook run, which nodes should be targeted, which packages must be installed during the deployment process, and where those packages should be obtained from:

* **`cloud`**: this parameter is used to indicate the target cloud for the deployment (either `aws` or `osp`); this controls both the role that is used to create new nodes (when a matching set of nodes does not exist in the target environment) and how the [build-app-host-groups](../roles/build-app-host-groups) role retrieves the list of target nodes for the deployment; if unspecified this parameter defaults to the `aws` value specified in the default configuration file
* **`tenant`**: this parameter is used to indicate the tenant name to use, either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; if unspecified this parameter defaults to the `datanexus` value specified in the default configuration file
* **`project`**: this parameter is used to indicate the project name to use, either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; if unspecified this parameter defaults to the `demo` value specified in the default configuration file
* **`dataflow`**: this parameter is used to indicate the dataflow name to use, either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; the dataflow tag is used to link together the clusters/ensembles (Cassandra, Zookeeper, Kafka, Solr, etc.) that are involved in a given dataflow; if this value is not specified, it defaults to a value of `none` during the playbook run
* **`domain`**: this parameter is used to indicate the domain name to use (eg. test, production, preprod), either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; if unspecified this parameter defaults to the `production` value specified in the default configuration file
* **`cluster`**: this parameter is used to indicate the cluster name to use, either when creating new nodes (when a matching set of nodes does not exist in the target environment) or when searching for a matching set of nodes in the [build-app-host-groups](../roles/build-app-host-groups) role; this value is used to differentiate clusters of the same type from each other when multiple clusters are deployed for a given application for the same tenant, project, dataflow, and domain; if this value is not specified it defaults to a value of `a` during the playbook run
* **`user`**: the username that should be used when connecting to the target nodes via SSH; the value for this parameter will likely change from one target environment to the next; if unspecified a value of `centos` will be used
* **`config_file`**: used to define the location of a *configuration file* (see the discussion of this topic, below); this file is a YAML file containing definitions for any of the configuration parameters that are described in this section and is more than likely a file that will be created to manage a given deployment. Storing the settings for a given deployment in such a file makes it easy to guarantee that all of the nodes in that deployment are configured consistently. If a value is not specified for this parameter then the default configuration file (the [config.yml](../config.yml) file) will be used; to override this behavior (and not load a configuration file of any kind), one can simply set the value of this parameter to `/dev/null` and specify all of the other, non-default parameters that are needed as extra variables during the playbook run
* **`private_key_path`**: used to define the directory where the private keys are maintained when the inventory for the playbook run is being managed dynamically; in these cases, the scripts used to retrieve the dynamic inventory information will return the names of the keys that should be used to access each node, and the playbook will search the directory specified by this parameter to find the corresponding key files. If this value is not specified then the current working directory will be searched for those keys by default

## Parameters used during the deployment process
These parameters are used to control the deployment process itself; currently this list consists of only one parameter (the `telegraf_package_list`), and that parameter is undefined by default.

* **`telegraf_package_list`**: the list of packages that should be installed on the targeted nodes; typically this parameter is left undefined, but it could be used to install additional packages on all targeted nodes during the playbook run
* **`telegraf_url`**: the URL where a custom version of the Telegraf executable can be downloaded from; if defined, the executable at this location will be downloaded to the target nodes and will overwrite the executable that is installed from the standard InfluxDB repository during the playbook run and, as such, could be used to replace that version with a newer (or older?) version during the deployment process
* **`local_telegraf_file`**: the local path (on the Ansible host) that points to a custom version of the Telegraf executable; if defined, the executable at this location will be uploaded to the target nodes and will overwrite the executable that is installed from the standard InfluxDB repository during the playbook run

## Parameters used to configure the Telegraf agents
These parameters are used configure the Telegraf agents themselves during a playbook run, defining things like the interfaces that they should be listening on for requests, the metrics/logs that they should be collecting, and the destinations that they should be reporting those metrics/logs to:

* **`internal_subnet`**: the CIDR block describing the subnet that any nodes being created by the playbook run should attach as a private network (`eth0`); this network is used for internode communications between the nodes of the clusters/ensembles that make up the dataflow being deployed; if it is not specified, then the default value of `10.10.1.0/24` from the [config.yml](../config.yml) file is used
* **`external_subnet`**: the CIDR block describing the subnet that any nodes being created by the playbook run should attach as a "public" network (`eth1`); this network is used to support client connections to the various services that make up the dataflow being deployed; if it is not specified, then the default value of `10.10.2.0/24` from the [config.yml](../config.yml) file is used
* **`configuration_list`**: a list of configuration parameters that should be used for the Telegraf agent itself and for each of the `measurement_sets` that the agent will be gathering and reporting to the Kafka cluster; for more information on this parameter see the detailed discussion of using this parameter to control the configuration of the Telegraf agents being deployed in the [Agent Configuration](Agent-Configuration.md) document.

## Determining interface names
The playbook in this repository will dynamically determine the names of the interfaces that correspond to the defined  `internal_subnet` and `external_subnet` CIDR block values and configure the Telegraf agents being deployed to communicate with the associated Kafka cluster over those interfaces. This is accomplished by dynamically constructing an `iface_description_array` parameter within the playbook, then using that parameter to determine the names of the corresponding interfaces and their IP addresses.

Put quite simply, the `iface_description_array` lets you specify a description for each of the networks that you are interested in, then retrieve the names of those networks on each machine in a variable that can be used elsewhere in the playbook. To accomplish this, the `iface_description_array` is defined as an array of hashes (one per interface), each of which include the following fields:

* **`type`**: the type of description being provided, currently only the `cidr` type is supported
* **`val`**: a value describing the network in question; since only `cidr` descriptions are currently supported, a CIDR value that looks something like `192.168.34.0/24` should be used for this field
* **`as_var`**: the name of the variable that you would like the interface name returned as

With these values in hand, the playbook will search the available networks on each machine and return a list of the interface names for each network that was described in the `iface_description_array` as the value of the fact named in the `as_var` field for that network's entry. For example, given this description:

```
    iface_description_array: [
        { as_var: 'data_iface', type: 'cidr', val: '192.168.34.0/24' },
        { as_var: 'api_iface', type: 'cidr', val: '192.168.44.0/24' },
    ]
```

In this example, the playbook will determine the name of the network that matches the CIDR blocks `192.168.34.0/24` and `192.168.44.0/24`, returning those interface names as the values of the `data_iface` and `api_iface` facts, respectively (eg. `eth0` and `eth1`). These two facts are then used later in the playbook to correctly configure the nodes to talk to each other (over the `data_iface` network) and listen on the proper interfaces for user requests (on the `api_iface` network).
