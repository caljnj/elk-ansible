# Intro

this playbook is essentially based on the discontinued official [ansbile-elastic](https://github.com/elastic/ansible-elasticsearch) playbook.

# Possible Issues

## node_types

in the inventory, every role must have a node_type. see xxxx for details

Coordinating nodes are supposed to be identified by a nonexistent `node.role` inside the elasticsearch.yml. 

So in  the inventory we denote coordinating nodes `node_types` as an empty yaml list:

    node_types: []

other roles are defined e.g. if a node has both master and hot for some weird reason it would have 

    node_types: 
      - master
      - hot

the backend processing of the jinja in elasticsearch.yml takes care of ONLY the case where the node_types list is populated. Then it spits out the values into the `node.roles` value. 

When `node_types` is undefined or is equal to the empty list, the node will be assigned the **coordinating role**.

i should probably get this to throw an error instead.

# Differences from the official playbook

## Intro 

There are some major differences here

## single playbook not playbook-per-server

### the change

the original ansible playbook made a quite weird design decision. 

Rather than having all the servers and server-vars in a single inventory, and run the playbook against all the servers - like you normally would. 

Instead they run the playbook once per server. 

This has many problems, but the primary one is that you cannot take advantage of inventory variables for stuff like: 

- list of all the master servers
- fqdns of all the hot nodes

So there was a lot of work to convert this from a per-server playbook to an all-servers-in-the-inventory playbook..

### the original config

so the original config ran the playbook in [this weird way](https://github.com/elastic/ansible-elasticsearch/blob/af05c6470ef63337deba7009eec6af3ea05e2193/README.md#L256-L293)

notice all the

- repetition of values
- values that would need to be manually harcoded e.g.:
    - `cluster.initial_master_nodes` - which if you use an inventory file, you can just pull these the list of master servers dynamically from the inventory.

```yaml
- hosts: master_node
  roles:
    - role: elastic.elasticsearch
  vars:
    es_heap_size: "1g"
    es_config:
      cluster.name: "test-cluster"
      cluster.initial_master_nodes: "elastic02"
      discovery.seed_hosts: "elastic02:9300"
      http.host: 0.0.0.0
      http.port: 9200
      node.data: false
      node.master: true
      transport.host: 0.0.0.0
      transport.port: 9300
      bootstrap.memory_lock: false
    es_plugins:
     - plugin: ingest-attachment

- hosts: data_node_1
  roles:
    - role: elastic.elasticsearch
  vars:
    es_data_dirs:
      - "/opt/elasticsearch"
    es_config:
      cluster.name: "test-cluster"
      cluster.initial_master_nodes: "elastic02"
      discovery.seed_hosts: "elastic02:9300"
      http.host: 0.0.0.0
      http.port: 9200
      node.data: true
      node.master: false
      transport.host: 0.0.0.0
      transport.port: 9300
      bootstrap.memory_lock: false
    es_plugins:
      - plugin: ingest-attachment
```

# Running the Play

## before you start

### Prepping the ansible server

install python

    dnf install python39

install ansible (dont use the the dnf packages they are _really_ behind)

    pip3.9 install --user ansible

from teh root run

    ansible-galaxy install -r requirements.yml

you'll get teh warning

>The scripts ansible, ansible-config, ansible-connection, ansible-console, ansible-doc, ansible-galaxy, ansible-inventory, ansible-playbook, ansible-pull and ansible-vault are installed in '/root/.local/bin' which is not on PATH.

so add it to the path on this machine, by adding this into `~/.bashrc`

    export PATH=$PATH:/root/.local/bin



collections are not installing using the requirements.yml.. so you need to install them manualy like this: 

    ansible-galaxy collection install community.general

### Variables that must be set

#### Encrypted User Passwords

the user needs to decide on these passwords: 

- elastic_root_password
- kibana_system_password
- logstash_system_password
- beats_system_password
- apm_system_password
- remote_monitoring_user_password

the cleartext password then needs to be encrypted [like this](https://docs.ansible.com/ansible/2.9/user_guide/vault.html#use-encrypt-string-to-create-encrypted-variables-to-embed-in-yaml): 

    ansible-vault encrypt_string --vault-password-file a_password_file 'foobar' --name 'the_secret'

then when running the playbook you need to specify the vault password with

    --ask-vault-password

### Licensing

* ```es_xpack_license``` - X-Pack license. The license is a json blob. Set the variable directly (possibly protected by Ansible vault) or from a file in the Ansible project on the control machine via a lookup:

```yaml
es_xpack_license: "{{ lookup('file', playbook_dir + '/files/' + es_cluster_name + '/license.json') }}"
```

If you don't have a license you can enable the 30-day trial by setting `es_xpack_trial` to `true`.

X-Pack configuration parameters can be added to the elasticsearch.yml file using the normal `es_config` parameter.

For a full example see [here](https://github.com/elastic/ansible-elasticsearch/blob/main/test/integration/xpack-upgrade.yml)

# Run it

    ANSIBLE_HOST_KEY_CHECKING=False ANSIBLE_FORKS=10 ansible-playbook -i inventories/test-inventory.yml site.yml --user=ec2-user --become --ask-vault-pass