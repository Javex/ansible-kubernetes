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

## Rolling out changes to nodes

There are some config changes that can disrupt your cluster and by default this
collection doesn't know what those are. As a result it's better to be careful:
When using a new version, make sure you have an `etcd` snapshot, but also update
each node one-by-one, ideally by first draining it. For workers this is fairly
simple: Follow [How to scale down a Talos cluster](https://www.talos.dev/v1.9/talos-guides/howto/scaling-down/).
Potentially, you might  want to also tell Longhorn to drain the node for storage
(via the UI). Once a node has been removed from the cluster, you can delete it
from Proxmox and let the playbook create a new one. To ensure any changed config
only ever applies to one node, you can specify the `talos_node_filter` variable.
If this variable is set, it will only apply the `talos` role to the node with
that name, for example `talos-prod-worker-1`. Here's what that looks like, for
example:

```bash
ansible-playbook k8s.yml --extra-vars=talos_node_filter=talos-prod-worker-1
```

If the node you are trying to replace is a controller node, make sure your
cluster remains in a healthy condition: If running in HA mode, do it one at a
time. If not in HA, create a second controller to join the cluster, then remove
the first one as a member using `talosctl etcd remove-member`.

In summary:

* If controller, remove from etcd
* Scale down node to remove it from cluster
* Delete VM in Proxmox
* Run playbook with node filter
* Repeat for next node
