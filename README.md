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

# Backup & Restore

This playbook automatically configures regular etcd snapshot backups to S3 using
restic. These backups are end-to-end encrypted with a passphrase you control.
Configure the variables with the right values before running the playbook. There
is currently no option to disable backups.

The backup solution is based on [this blog post](https://alexlubbock.com/encrypted-backup-talos-etcd-aws-s3)
and [repo](https://github.com/alubbock/talos-etcd-backup). It explains the setup
in more details.

## Repo Init

After the playbook has finished running you need to initialise the restic repo
manually. To do this, export environment variables:

```bash
export RESTIC_PASSWORD='...'
export AWS_SECRET_ACCESS_KEY='...'
export AWS_ACCESS_KEY_ID='...'
export RESTIC_REPOSITORY='s3:'
```

Take these values from the playbook. Note that here you **do** need the `s3:`
prefix for the bucket. Once you have all variables set run:

```bash
restic init
```

That should show that the repo has been initialised. Now backups should work
which you can test with the following command:

```bash
kubectl -n kube-system create job --from=cronjob/etcd-backup etcd-backup-first
```

See the [blog post](https://alexlubbock.com/encrypted-backup-talos-etcd-aws-s3)
for more details. Once you have confirmed that you have a backup, it's time to
test the restore process.

## Restore

Most importantly: You should test that restoring from backup works. These
instructions will also help in case you need to restore for real. Refer to
[Talos Disaster Recovery](https://www.talos.dev/v1.9/advanced/disaster-recovery/)
for more details. The steps below are specific to this playbook.

* Download a backup onto your system: `restic restore latest --target tmp/etcd-restore`
* Check that there is a `etcd.snapshot` file in the `tmp/etcd-restore` folder
* Go into Proxmox and take a snapshot of every VM in case the restore process
  fails. You can roll back the entire cluster. At least take a snapshot of the
  control plane nodes.
* Stop all control plane nodes in Proxmox. Use "Stop" not "Shutdown" to simulate
  spontaneous hardware failure.
* Remove all tags from the stopped nodes to exclude them from the playbook
  without deleting them.
* Rename the nodes to something like "backup-talos-prod-controller-1" etc.
* Set the `talos_etcd_recovery_snapshot` variable in `group_vars/k8s.yml` to the
  **absolute** path of the snapshot (find it with `readlink -f`) if you're not
  sure.
* Run the playbook again. If it fails run it again, provisioning with a static
  IP address can cause the run to fail.

It should complete successfully and `kubectl get nodes` should report all nodes
as ready. Congratulations, you've restored from backup!
