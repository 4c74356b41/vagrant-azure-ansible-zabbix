# Project overview

Installs several ubuntu VM's on Azure (master and slaves)  
Installs redis server, dotnetcore, zabbix server [master]  
Installs zabbix-agent, pip, redis python module [slaves]  
Copies fake app payload to /home/vagrant those VM's (payloads have to be invoked manually)

# Project structure

```
-|
 |-ansible     # Contains all the ansible playbooks
 |-artifacts   # Contains payload and configuration
 -vagrantfile  # Vagrantfile to deploy this project
```

# Prerequisites

### Azure related
- Working Azure Subscription
- Azure AD Application with rights to provision resources
- Environment variables

##### Configure environment variables for this to work

to make Azure Provider work with vagrant you have to create following environment variables:
- tenant_id: Your Azure Active Directory Tenant Id.
- client_id: Your Azure Active Directory application client id.
- client_secret: Your Azure Active Directory application client secret.
- subscription_id: The Azure subscription Id you'd like to use.

For instructions on how to setup an Azure Active Directory Application see: https://azure.microsoft.com/en-us/documentation/articles/resource-group-create-service-principal-portal/

### Host related
- [vagrant](https://www.vagrantup.com)
- [ansible](https://www.ansible.com)
- [zabbix-api](https://pypi.python.org/pypi/zabbix-api) (python package)
- [vagrant-azure](https://github.com/Azure/vagrant-azure) (vagrant plugin)
- vagrant-azure base box
- [dj-wasabi.zabbix-server](https://github.com/dj-wasabi/ansible-zabbix-server), [dj-wasabi.zabbix-agent](https://github.com/dj-wasabi/ansible-zabbix-agent), [geerlingguy.mysql](https://github.com/geerlingguy/ansible-role-mysql) (ansible-galaxy roles)

##### Configure your host

```
apt-get install vagrant
apt-get install ansible
pip install zabbix-api
vagrant plugin install vagrant-azure --plugin-version '2.0.0.pre6'
vagrant box add azure https://github.com/azure/vagrant-azure/raw/v2.0/dummy.box
ansible-galaxy install geerlingguy.mysql
ansible-galaxy install dj-wasabi.zabbix-server
ansible-galaxy install dj-wasabi.zabbix-agent
```

# Working with project

Vagrantfile should be configured to adjust your configuration:

- config.ssh.private_key_path - should contain path to your ssh key
- should your ssh key need a passphrase you can use ssh-agent to provide it
- to modify boxes configurations you should modify azure.json ([reference](https://github.com/Azure/vagrant-azure/blob/v2.0/README.md))

### Run project

navigate to project directory and run:

```
vagrant up --no-provision
vagrant --count=1 up
```

> this will probably work properly with just `vagrant --count=1 up` on "normal" providers but due to how vagrant-azure
> is implemented its not possible to run several concurrent provisioning operations. So consider adding nodes one by one. Occasionally it works when you add several nodes at once, but rarely.

Should you need more consumer nodes, you can run
```
vagrant --count=2 up
vagrant --count=3 up
```

To access Zabbix you should navigate to the master node ip or dns (http://ip/zabbix), user: Admin, pass: zabbix. Can be obtained from Azure portal or ansible configuration data.

### Message generator

To run message generator go to `/home/vagrant/dotNetCore` and do `dotnet run 1` (on master node, you can use any integer, this will be passed to channel and worker will sleep between messages for this amount of time). Must be root.

```
vagrant ssh
sudo su
cd dotNetCore/
dotnet run 1
```

Messages are sent every 200ms, so to clog the queue, just use any integer greater than 0.

### Message consumer

To run message consumer go to `/home/vagrant/pythonListener` and do `./_.py` (on slave nodes)

```
vagrant --count=x ssh slave№
# for example to go to first slave out of 3 slaves do
# vagrant --count=3 ssh slave0

sudo su
cd pythonListener/
./_.py
```

# Known issues

1. I've encountered this [bug](https://github.com/dj-wasabi/ansible-zabbix-server/issues/43), which seems to be fixed, but my package didn't seem to include that. [Fix](https://github.com/dj-wasabi/ansible-zabbix-server/commit/1251c6c4c2f9357b589b21b1dcd08b9645deae41#commitcomment-20949317) is in the `tasks/Debian.yml` file.
2. Sometimes ansible runs 2 times, didn't yet have time to fight this, doesn't really break things, but is very annoying
3. You should use the same ssh key for all the machines ([Reference](http://blog.wjlr.org.uk/2014/12/30/multi-machine-vagrant-ansible-gotcha.html))
4. Sometimes ansible provisioner will fail with error like "%FQDN% is too long for Unix socket". No real work around at this point. Vagrant generated random FQDN for Azure VM's so we can't control that. Recreate environment fixes the issue.