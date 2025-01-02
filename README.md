# Kubernetes Ansible Roles

This collection of roles creates a Kubernetes cluster from scratch using
Proxmox to provision VMs and Talos as the host OS for each node. It then
creates a bunch of resources on the cluster to provide services for a typical
cluster. If the setup described here fits your needs, feel free to deploy it or
copy it and make changes.

The recommended way is to install this repo as a collection:

```bash
ansible-galaxy collection install https://github.com/Javex/ansible-kubernetes
```

Create an inventory file from the example. Make sure your
`ansible.cfg` is to configured to look for this file, e.g. within a folder
called `inventory/`. Edit the file, adjust variables as necessary.

```bash
cp examples/02-hosts.proxmox.yml inventory/
```

Create the group variables and adjust those variables as well:

```bash
mkdir -p group_vars/
cp examples/group_vars_k8s.yml group_vars/k8s.yml
```

Finally, create a playbook from the example:

```bash
cp examples/playbook.yml k8s.yml
```

Edit this file, too. Now run the playbook:

```bash
ansible-playbook k8s.yml
```
