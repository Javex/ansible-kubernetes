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

Create an inventory file:

```yaml
plugin: community.general.proxmox
url: https://pve.example.com:8006
user: ansible@pam
token_id: playbook
# Create your own encrypted proxmox token
token_secret: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  3931396.....63316135373930

# Needed for filtering and composing, e.g. getting tags
want_facts: true

# Nodes are currently in static inventory, so don't fetch them again
# If changing this, pay attention to the "want_proxmox_nodes_ansible_host"
# option.
exclude_nodes: true

# It's better to fail to enforce well-written expressions in filters etc. If it
# fails we can fix it.
strict: true

# At the moment only the below VMs are managed via dynamic inventory, the rest
# is managed statically.
filters:
  - '"k8s" in (proxmox_tags_parsed|default([]))'

# Note: Update the subnet below to match whatever subnet your IP addresses will
# be assigned in.
compose:
  ansible_host: >
    proxmox_agent_interfaces |
    default([]) |
    map(attribute="ip-addresses") |
    flatten |
    ansible.utils.ipaddr('192.168.1.0/24') |
    ansible.utils.ipaddr('address') |
    first |
    default()
```

Then create a playbook like this:

```yaml
---
- name: Download Talos image onto proxmox hosts
  hosts: proxmox
  become: true
  tasks:
    - name: Download image
      ansible.builtin.get_url:
        url: https://factory.talos.dev/image/{{ talos_image_id }}/v{{ talos_image_version }}/metal-amd64.iso
        dest: /var/lib/vz/template/iso/talos-metal-amd64.iso
        checksum: "{{ talos_image_hash }}"
        owner: root
        group: root
        mode: "0644"
  vars:
    talos_image_id: ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
    talos_image_version: 1.8.4
    talos_image_hash: sha256:c26ede0e7ffd578d79c7afb515b0eb7da5bb32b1a0e142813726473760faa060

- name: Prepare kubernetes cluster
  hosts: localhost
  gather_facts: false

  pre_tasks:
    - name: Determine current working directory
      ansible.builtin.command: pwd
      register: pwd_cmd

    - name: Create temp path from cwd
      ansible.builtin.set_fact:
        tmp_path: "{{ pwd_cmd.stdout }}/tmp"
        cwd: "{{ pwd_cmd.stdout }}"

    - name: Include Proxmox login task to get facts with credentials
      ansible.builtin.include_role:
        name: proxmox
        tasks_from: login.yaml

- name: Include playbook for javex.kubernetes collection
  ansible.builtin.import_playbook: javex.kubernetes.k8s
  vars:
    # Vars used in multiple roles
    dns_ips:
      - 192.168.1.20
      - 192.168.1.21
      - 192.168.1.22
    cluster_ip: 10.96.0.10

    # Talos specific vars
    talos_proxmox_node: pve
    talos_vm_id: 201
    talos_vm_name: talos-prod-control-1
    # This matches both pve & pve2 as pve2 runs a 3rd gen Sandy Bridge
    # IBRS offers spectre protection
    talos_vm_cpu: SandyBridge-IBRS
    talos_tmp_base_path: "{{ tmp_path }}"
    talos_proxmox_user: "{{ proxmox.user }}"
    talos_proxmox_token_id: "{{ proxmox.token_id }}"
    talos_proxmox_token_secret: "{{ proxmox.secret }}"
    talos_proxmox_host: "{{ proxmox.host }}"
    talos_ctl_version: v1.9.0
    talos_cluster_name: homelab-prod
    talos_secrets_file: "{{ cwd }}/files/talos/secrets.prod.enc.yaml"
    talos_image_id: ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515
    talos_image_version: 1.8.4
    talos_kubernetes_version: 1.31.4
    # Issuer URL as provided by IdP
    talos_oidc_issuer_url: https://auth.example.com/application/o/kubernetes/
    # Client ID (the contents of the "aud" claim)
    talos_oidc_client_id: HbpLcSurg2G4OIQi7OneGiWfa0JbPv9rM8TWBGTA
    # Name of claim in JWT for groups
    talos_oidc_groups_claim: groups

    # MetalLB specific vars
    # Sharing space with local DHCP server that reserves .100-.255
    # Plus lots of IPs already assigned below .60, but sparse
    # Can always list more IPs here as needed.
    metallb_loadbalancer_ips: 192.168.1.60-192.168.1.80

    # CoreDNS specific vars
    coredns_cluster_domain_internal: "k8s.example.com"
    coredns_cluster_domain_external: "dev.example.com"

    # cert-manager specific vars
    cert_manager_acme_email: "me@example.com"
    # Create your own encrypted vault secret here
    cert_manager_cloudflare_api_token: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      61.....8

    # Traefik specific vars
    traefik_dns_names:
      - "*.example.com"

    # kubernetes dashboars specific vars
    kubernetes_dashboard_hostnames:
      - k8s-dashboard.example.com
```

Now run the playbook:

```bash
ansible-playbook myplaybook.yml
```
