Ansible-Graylog-Cluster
=========

#### It is highly recommended that you do not directly access this repository from your CI/CD jobs. Make a fork in your project's SCM instead. By doing so, you make sure that your cluster will not be broken in case one of the new commits appear to have a bug. You can keep the fork synchronized with the origin manually

Read these before installig Elasticsearch and MongoDB:

* https://docs.graylog.org/ >>> Installing Graylog

* https://docs.graylog.org/ >>> Configuring Graylog >>> Elasticsearch

Read this before configuring Graylog URIs:

* https://docs.graylog.org/ >>> Configuring Graylog >>> server.conf >>> Web & REST API

```
docker run -dit --name=mongo --network=elasticsearch -e MONGO_INITDB_ROOT_USERNAME="root" -e ME_CONFIG_MONGODB_ADMINUSERNAME="root" -e ME_CONFIG_MONGODB_ADMINPASSWORD="io7NzuF9RdSdFkmIuBPC" -e MONGO_INITDB_ROOT_PASSWORD="io7NzuF9RdSdFkmIuBPC" -v /srv/mongodb/db:/data/db -p 0.0.0.0:27017:27017 mongo:4.0
```

or

```
docker run -dit --name=mongo --network=elasticsearch -v /srv/mongodb/db:/data/db -p 0.0.0.0:27017:27017 mongo:4.0
```

### Config Examples

* `FILE: {{ playbook_dir}}/vars/elasticsearch-secrets-graylog-production.yml`

```
elasticsearch_password: <STRONG_PASSWORD_FOR_ELASTICSEARCH>

elasticsearch_root_ca_cert_base64: <ROOT_CA_CERT_BASE64_ENCODED>
elasticsearch_root_ca_key_base64: <ROOT_CA_KEY_BASE64_ENCODED>
```

#### See https://github.com/arsenii-stefanov/ansible-elasticsearch-cluster for a playbook example and complete Elasticsearch installation guide

* `FILE: {{ playbook_dir}}/vars/elasticsearch-graylog-production.yml`

```
########################################
##     START ELASTICSEARCH CONFIG     ##
########################################
elasticsearch_installation_type: "docker"        # Available options: docker

elasticsearch_main_mount_dir: "/srv"

elasticsearch_disk_mounts: [
    {
      src: "/dev/sdb",
      dest: "{{ elasticsearch_main_mount_dir }}",
      fstype: "ext4",
      fs_opts: "-m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard",
      mount_opts: "discard,defaults,nofail",
      state: mounted
    }
]

elasticsearch_cluster_name: "elasticsearch-graylog-production-cluster"

elasticsearch_version: "6.6.1"

# openssl req -x509 -newkey rsa:4096 -keyout root-ca-key.pem -out root-ca-cert.pem -days 3650 -nodes -subj "/C=US/ST=NY/L=NewYork/O=MyCompany/OU=IT/CN=MyDomain/emailAddress=admin@example.com"

elasticsearch_openssl_csr:
  common_name: "Elasticsearch"
  country_name: "US"
  state_or_province_name: "NY"
  locality_name: "NY"
  organizational_unit_name: "IT"
  organization_name: "Elasticsearch"
  email_address: "admin@elasticsearch.local"
  subject_alt_name:
    - "DNS:{{ inventory_hostname }}"
    - "IP:{{ ansible_host }}"

# You may add env vars here
elasticsearch_env_vars:
  # JVM heap size should NOT exceed 50% of your physical RAM
  ES_JAVA_OPTS: "-Xms8g -Xmx8g"
  MAX_LOCKED_MEMORY: "unlimited"
  ELASTIC_PASSWORD: "{{ elasticsearch_password }}" # 'elasticsearch_password' should be defined in an Ansible secrets file

# These settings will be used in Jinja2 templates
elasticsearch_config_opts:
  number_of_shards: "2"
  number_of_replicas: "0"
  backup_repo: ""
  minimum_master_nodes: "2" # This setting is required if you use an Elasticsearch version older than 7.x

# The entire dictionary will be added to the 'xpack' dictionary in the 'templates/elasticsearch/elasticsearch.yml.j2' as is
elasticsearch_xpack_config:
  license:
    self_generated:
      type: "basic"
  monitoring:
    collection:
      enabled: true
  security:
    enabled: true
### SSL certs are not required for Elasticsearch 6.x.x
#    transport:
#      ssl:
#        enabled: true
#        verification_mode: "certificate"
#        certificate_authorities: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_root_ca_cert_name }}"
#        certificate: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_node_cert_name }}"
#        key: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_node_key_name }}"
#    http:
#      ssl:
#        enabled: false
#        certificate_authorities: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_root_ca_cert_name }}"
#        certificate: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_node_cert_name }}"
#        key: "{{ elasticsearch_certs_container_dir }}/{{ elasticsearch_node_key_name }}"

######################################
##     END ELASTICSEARCH CONFIG     ##
######################################

#############################################
###                DOCKER                 ###
#############################################
docker_packages_ubuntu: [ "docker-ce" ]

#############################################
###                PYTHON                 ###
#############################################

ansible_python_interpreter: /usr/bin/python3
python_packages: [ python3-pip, python3-numpy, python3-scipy ]

python_modules_docker: [
  { name: "docker", state: "latest", extra_args: "--user", executable: "pip3" }
]
```

* `FILE: {{ playbook_dir}}/vars/graylog-secrets-production.yml`

```
graylog_secret_vars:
  # cat /dev/urandom | tr -dc a-zA-Z0-9 | fold -w 96 | head -n 1
  pass_secret: "<SECRET_STRING_1>"
  # PWD=$(cat /dev/urandom | tr -dc a-zA-Z0-9 | fold -w 30 | head -n 1); SHA=$(echo -n ${PWD} | sha256sum | awk '{print $1}'); echo "Password: ${PWD}"; echo "SHA2: ${SHA}"
  root_pass_sha2: "SECRET_STRING_2"
```

* `FILE: {{ playbook_dir}}/vars/graylog-cluster-production.yml`

```
---

# START GRAYLOG CONFIG
graylog_installation_type: "docker"        # Available options: docker

graylog_dns_names: [ "graylog.example.local" ]

graylog_config:
  docker_opts:
    container_name: "graylog"
    image_name: "graylog/graylog"
    image_tag: "3.0.0"
    ports:
      - "0.0.0.0:{{ graylog_http_port }}:{{ graylog_http_port }}"
      - "0.0.0.0:12201-12205:12201-12205"
      - "0.0.0.0:12301-12310:12301-12310/udp"
    env_vars:
      GRAYLOG_PASSWORD_SECRET: "{{ graylog_secret_vars['pass_secret'] }}"
      GRAYLOG_ROOT_PASSWORD_SHA2: "{{ graylog_secret_vars['root_pass_sha2'] }}"
      GRAYLOG_IS_MASTER: "{{ hostvars[inventory_hostname]['graylog_master'] | default('false')}}"
      GRAYLOG_SERVER_JAVA_OPTS: "-Xms2g -Xmx2g"
      GRAYLOG_HTTP_BIND_ADDRESS: "0.0.0.0:{{ graylog_http_port }}"
      GRAYLOG_HTTP_EXTERNAL_URI: "http://0.0.0.0:{{ graylog_http_port }}/"
      GRAYLOG_HTTP_PUBLISH_URI: "http://{{ ansible_host }}:{{ graylog_http_port }}/"
      GRAYLOG_ELASTICSEARCH_HOSTS: "http://{{ elasticsearch_user | default('elastic') }}:{{ elasticsearch_password }}@10.236.0.130:{{ graylog_es_http_port }},http://{{ elasticsearch_user | default('elast$
      GRAYLOG_ELASTICSEARCH_DISCOVERY_ENABLED: "true"
      GRAYLOG_MONGODB_URI: "mongodb://mongodb-graylog.example.local:27017/graylog"
      GRAYLOG_ROOT_TIMEZONE: "GMT"
      GRAYLOG_ELASTICSEARCH_MAX_TIME_PER_INDEX: "1d"
      GRAYLOG_ELASTICSEARCH_SHARDS: "2"
      GRAYLOG_ELASTICSEARCH_MAX_NUMBER_OF_INDICES: "60"
      GRAYLOG_ROTATION_STRATEGY: "time"
    log_driver: "json-file"
    network_mode: "bridge"
    networks: "{{ graylog_docker_network | default('elasticsearch') }}"
    purge_networks: "false"
  config_opts:
    empty: ""

# END GRAYLOG CONFIG

# START NGINX CONFIG

install_nginx_reverse_proxy: true     # whether to install NGINX as a reverse proxy for Graylog or not 

graylog_nginx_docker_image_tag: "1.17-alpine"

# END NGINX CONFIG
```

* `FILE: {{ playbook_dir }}/requirements/graylog-requirements.yml`

```
---
# Basic server setup
- name: ansible-basic-server-setup
  src: git@github.com:arsenii-stefanov/ansible-basic-server-setup.git
  scm: git
  version: master
# Set up Python
- name: ansible-python
  src: git@github.com:arsenii-stefanov/ansible-python.git
  scm: git
  version: master
# Set up Docker
- name: ansible-docker
  src: git@github.com:arsenii-stefanov/ansible-docker.git
  scm: git
  version: master
# Set up Elasticsearch
- name: ansible-elasticsearch-cluster
  src: git@github.com:arsenii-stefanov/ansible-elasticsearch-cluster.git
  scm: git
  version: master
```

* `FILE: {{ playbook_dir }}/playbook.yml`

```
- hosts: graylog_cluster_production
  connection: ssh
  become: true
  become_user: root
  become_method: sudo
  any_errors_fatal: true
  gather_facts: true
  serial: 1

  pre_tasks:

    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "elasticsearch-graylog-production.yml"
        - "elasticsearch-secrets-graylog-production.yml"

  roles:
    - { role: "ansible-basic-server-setup", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - { role: "ansible-python", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - { role: "ansible-docker", when: skip_basic_server_setup is not defined or ( skip_basic_server_setup is defined and skip_basic_server_setup == false ) }
    - ansible-elasticsearch-cluster

- hosts: graylog_cluster_production
  connection: ssh
  become: true
  become_user: root
  become_method: sudo
  any_errors_fatal: true
  gather_facts: true

  pre_tasks:

    - name: Include Variables
      include_vars: "{{ item }}"
      with_items:
        - "graylog-secrets-production.yml"
        - "graylog-cluster-production.yml"

  roles:
    - ansible-graylog-cluster
```

* `FILE: {{ playbook_dir }}/inventory-graylog_cluster_production`

```
[graylog_cluster_production]
graylog-node-1 ansible_host=10.236.0.130 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=master-data graylog_master=true
graylog-node-2 ansible_host=10.236.0.131 ansible_user=deploy-bot-infra ansible_port=22 es_node_role=master-data
```

#### Run Ansible

* Install Ansible Galaxy requirements

```
ansible-galaxy -vvvv install -f -r requirements/graylog-requirements.yml
```
 * Export  the Ansible Vault password
```
export VAULT_PASS=<Vault password>
```

* Perform initial setup of an Elasticsearch cluster (with `elasticsearch_initial_setup: true` your ES will not be restarted upon updating the config files)

```
ansible-playbook --vault-password-file ./vault.py -i inventory-graylog_cluster_production graylog-cluster-production.yml -e '{"elasticsearch_initial_setup": true}'
```

* If you need to update some config files, directories, OS settings, add new nodes or you would like to insure your cluster integrity, you can use this command. In case any config file is updated, your E$

```
ansible-playbook --vault-password-file ./vault.py -i inventory-graylog_cluster_production graylog-cluster-production.yml -e '{"skip_basic_server_setup": true}'
```
