# BrokerLathe
BrokerLathe is a systems deployment and management project to deploy a highly available message broker with Apache Apollo and Hadoop on FreeBSD. It came about after researching message brokers with mission critical high volume message broker deployment and scaling needs.

# Approach
Using Ansible playbooks and roles, configuration management and source controlled code changes, we wire several separate projects together to create a highly available message broker system running on FreeBSD nodes or jails.

# Playbooks

Broker Deployment

```bash
ansible-playbook -K -i inventory/brokerlathe_local.ini configure_broker.yml -e "designation=local" -v
```
If you are doing this with a local jail inventory, you need to run the playbook as root with sudo because jails.

If the playbook completes successfully, Apollo should now be running on the inventory IP at default admin port of 61680, e.g. http://10.51.4.133:61680/
