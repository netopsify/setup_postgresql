# Setting up Postgresql DB on Centos host

Though these steps can be used to deploy/configure postgresql cluster on any linux platform due the increadible roles built by anx team.
I recently had a need to setup on Centos with version 9.6 and had to make some changes to it. Hence it provides a good reference for Centos 7.x.

## Pre-requisites

### Ansible Controller

1. Ansible is installed on the host where you want to run ansible.
2. `inventory` file is created with the required postgresql servers.
3. Ensure that Ansible controller can reach to the postgresql servers using their hostname. *Hostnames are important in replication setting later on*
4. Connectivity b/w ansible controller and Centos host is established.
5. Ensure all machines are reacheable through hostnames.
6. Run this command to verify `ansible -i inventory servers -m ping`, `ansible -i inventory postgresql1 -m ping`
7. You can also test the facts collection using `ansible -i inventory servers-m setup`, `ansible -i inventory postgresql1-m setup`

## Configuration for replication

- Download the following role: `https://github.com/Demonware/postgresql.git` into `./roles/postgresql` by running `git clone https://github.com/Demonware/postgresql.git roles/postgresql`
- checkout to replication branch example `cd roles/postgresql && git checkout add-repmgr-extension`
- Copy all variables from `./roles/postgresql/defaults/repmgr.yml` to `./roles/postgresql/defaults/main.yml`
- Update `vars/main.yml` with replication settings. Example given below.
- Open `roles/postgresql/tasks/extensions/repmgr.yml` file and change the line 15 to `name: "repmgr{{postgresql_version | replace('.', '')}}"`

### Running Ansible playbook

- Create ansible playbook using the ansible role
- If you would like to change the default vars of the role you can create a custom `vars` file and place all your vars in it. I have created `./vars/main.yml`
- Create inventory file containing DB servers
- Ensure the hosts are reachable via names
- Run the ansible playbook using `ansible-playbook -i inventory pb_postgres.yml`

*Ansible Playbook*

```ansible
#pb_postgres.yml
---
- hosts: db
  become: yes
  pre_tasks:
    - name: Include Vars
      include_vars: vars/main.yml

    - name: Change hostname
      hostname:
        name: "{{ inventory_hostname }}"

    - name: Open firewall ports for postgresql
      firewalld:
        port: "{{ postgresql_port }}/tcp"
        permanent: yes
        state: enabled

    - name: Bounce firewalld
      service:
        name: firewalld
        state: restarted
  roles:
    - role: holms.fqdn
    - role: postgresql
```

*Vars file*

```ansible
# ./vars/main.yml
---
# Basic settings
postgresql_version: 9.6
postgresql_encoding: "UTF-8"
postgresql_locale: "en_US.UTF-8"
postgresql_ctype: "en_US.UTF-8"

postgresql_admin_user: "postgres"
postgresql_default_auth_method: "peer"

postgresql_service_enabled: true # should the service be enabled, default is true

postgresql_cluster_name: "main"
postgresql_cluster_reset: false

# List of databases to be created (optional)
# Note: for more flexibility with extensions use the postgresql_database_extensions setting.
postgresql_databases:
  - name: jiradb
    owner: jira          # optional; specify the owner of the database
    encoding: "UTF-8"   # override global {{ postgresql_encoding }} variable per database
  - name: "{{repmgr_database}}"
    owner: "{{repmgr_user}}"
    encoding: "UTF-8"

# List of users to be created (optional)
postgresql_users:
  - name: jira
    pass: password
    encrypted: no  # if password should be encrypted, postgresql >= 10 does only accepts encrypted passwords
  - name: "{{repmgr_user}}"
    pass: password
    encrypted: yes  # if password should be encrypted, postgresql >= 10 does only accepts encrypted passwords

# List of user privileges to be applied (optional)
postgresql_user_privileges:
  - name: jira                   # user name
    db: jiradb                  # database
    priv: "ALL"                 # privilege string format: example: INSERT,UPDATE/table:SELECT/anothertable:ALL
    role_attr_flags: "CREATEDB" # role attribute flags
  - name: "{{repmgr_user}}"
    db: "{{repmgr_database}}"
    priv: "ALL"
    role_attr_flags: "SUPERUSER,REPLICATION"

# Manage replication with repmgr (optional)
repmgr_version: "repmgr96"
repmgr_target_group: "postgres_cluster" # Repmgr works on the hosts defined under the group "postgres_cluster" defined in the inventory file
repmgr_target_group_hosts: "{{ groups[repmgr_target_group] }}"
repmgr_master: "postgresql1" # Host postgresql1 will be the PostgreSQL primary instance, postgresql2 and postgresql3 will replicate from postgresql1

postgresql_ext_install_repmgr: yes # Installs repmgr extension on the PostgreSQL cluster
repmgr_user: repmgr
repmgr_database: repmgr

# Modifying the PostgreSQL configuration to enable replication.
# We will modify the parameters like wal_level, max_wal_senders, max_replication_slots, hot_standby, archive_mode, archive_command
postgresql_wal_level: "replica"
postgresql_max_wal_senders: 10
postgresql_max_replication_slots: 10
postgresql_wal_keep_segments: 100
postgresql_hot_standby: on
postgresql_archive_mode: on
postgresql_archive_command: "test ! -f /tmp/%f && cp %p /tmp/%f"
postgresql_shared_preload_libraries:
  - repmgr

# Allowing External Hosts to Connect to the PostgreSQL Server
postgresql_pg_hba_custom:
  - {type: "host", database: "jiradb", user: "jira", address: "0.0.0.0/0", method: "md5" }
# For replication
  - { type: "host", database: "all", user: "all", address: "192.168.189.0/24", method: "md5" }
  - { type: "host", database: "replication", user: "repmgr", address: "192.168.189.0/24", method: "md5" }  
  - { type: "host", database: "replication", user: "repmgr", address: "127.0.0.1/32", method: "md5" }  

postgresql_listen_addresses:
  - "*"
postgresql_port: 5432

```

*inventory file*

```ansible
# inventory
postgresql1 ansible_host=192.168.189.151 ansible_user=root ansible_password=root ansible_become=true
postgresql2 ansible_host=192.168.189.152 ansible_user=root ansible_password=root ansible_become=true
postgresql3 ansible_host=192.168.189.153 ansible_user=root ansible_password=root ansible_become=true

[servers]
postgresql1
postgresql2
postgresql3

[db:children]
servers

[postgres_cluster:children]
servers
```

### Verification

- Verify service is running using `systemctl status postgresql-9.6`
- Log into *psql* and run show command.
  - `su - postgres`
  - `\psql`
  - `show listen_addresses ;`
  - list users `\du`
  - list DBs `\l`
  - `\q`
  - `/usr/pgsql-9.6/bin/repmg -f /etc/postgresql/9.6/data/repmgr.conf cluster show`

### Troubleshooting machines

1. Ensure network interfaces are enabled by going to `/etc/sysconfig/network-scripts/` and looking up the interface script. In my case its `/etc/sysconfig/network-scripts/ifcfg-ens33` and enable `ONBOOT` setting to `yes`
2. If its a fresh VM ensure you install the following packages:
    1. `openssh-server` using `yum install -y openssh-server`
3. Enable SSHD by running `systemctl enable sshd` and `systemctl start sshd`. Finally check the status by running `systemctl status sshd`
4. Either disable firewall or open ports for remote connectivity.
    1. Disable - You can disable the firewall by running `systemctl disable firewalld` and `systemctl stop firewalld`. Its better to reboot VM after this.
    2. Allow Ports - Run `firewall-cmd --list-all`
5. If the cluster is not coming up ensure you have `postgresql_listen_addresses` variable is defined with the required IPs. You can test it with setting it to below.

```yaml
postgresql_listen_addresses:
  - "*"
```

## References

[PostgreSQL Deployment and Maintenance with Ansible](https://severalnines.com/database-blog/postgresql-deployment-and-maintenance-ansible)

[PostgreSQL Replication Setup &amp; Maintenance Using Ansible](https://severalnines.com/database-blog/postgresql-replication-setup-maintenance-using-ansible)

[Demonware/postgresql](https://github.com/Demonware/postgresql)

[ANXS/postgresql](https://github.com/ANXS/postgresql)