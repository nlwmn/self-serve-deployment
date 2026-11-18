# Ansible Automation Portal self-serve deployment

This deploys the Ansible automation portal appliance as a KVM guest on an existing RHEL host. Ansible runs from this repo against that host. The host is registered to Red Hat, the portal disk image is copied into libvirt, and cloud-init supplies the portal admin user, an SSH public key, and the Automation Controller connection. Host port 443 is forwarded to the guest.

## Layout

| Path | Purpose |
| --- | --- |
| `deploy_self_serve_vm.yml` | Register the host, then create the portal VM |
| `download_image.yml` | Download the portal qcow2 from Red Hat |
| `destroy_self_server_vm.yml` | Stop and remove the guest |
| `inventory/self-serve-inventory.yml` | The KVM host (`selfservehost`). Gitignored. Create it from the example |
| `vars/vars_self_serve.yml` | Image path, VM size, and portal URL |
| `vars/vault_private_vars.yml` | Secrets. Gitignored. Create it from the example |
| `templates/user-data.j2` | Cloud-init config written into the guest |
| `templates/meta-data.j2` | Cloud-init instance id and hostname |

`ansible.cfg` sets the default inventory to `inventory/self-serve-inventory.yml`.

## Before you run anything

1. Copy the vault template and replace every placeholder:

   ```bash
   cp vars/vault_private_vars.yml.example vars/vault_private_vars.yml
   ```

2. Copy the inventory template and set `ansible_host` and `ansible_user` for the KVM host. The user needs sudo. Put the SSH key in `~/.ssh/config`, or uncomment `ansible_ssh_private_key_file` in the copy.

   ```bash
   cp inventory/self-serve-inventory.yml.example inventory/self-serve-inventory.yml
   ```

3. On the control node, install the libvirt collection used by the destroy playbook:

   ```bash
   ansible-galaxy collection install community.libvirt
   ```

   The KVM host needs `python3-libvirt` for that module. The deploy playbook installs `qemu-kvm`, `libvirt`, `virt-install`, and `genisoimage` itself.

## Non-secret settings

`vars/vars_self_serve.yml` holds the image checksum, download directory, qcow2 path, VM name, memory, CPU count, and the interface used for the port forward (`portal_forward_iface`, default `eth0`).

`portal_user` is the guest admin account. `database_type` is `builtin`. `portal_base_url` is `https://` plus the KVM host address from the inventory. That URL is what users open in a browser.

## Secrets

`vars/vault_private_vars.yml` is listed in `.gitignore`. Keep it that way.

| Variable | Used by |
| --- | --- |
| `ssh_public_key` | Cloud-init, as an authorized key for `portal_user` |
| `aap_host_url` | Controller URL written into the guest |
| `aap_api_token` | Controller API token |
| `aap_check_ssl` | TLS verification for that controller |
| `aap_oauth_client_id` | OAuth client on the controller |
| `aap_oauth_client_secret` | OAuth client secret |
| `redhat_offline_token` | `download_image.yml` only |
| `rhsm_org` | Host registration |
| `rhsm_activation_key` | Host registration |

### `aap_check_ssl`

This boolean is rendered into the guest cloud-init config as `aap.check_ssl`. `true` makes the portal verify the Automation Controller certificate. Leave it `true` when the controller certificate is publicly trusted or the guest has the right CA. Set it `false` only when the controller uses a certificate the guest cannot validate, such as a short-lived lab cert. `false` disables TLS verification for that connection.

## Playbooks

Run them from the repo root, in this order.

### 1. `download_image.yml`

Downloads the portal disk image onto the KVM host.

- `image_sha256` in `vars/vars_self_serve.yml` must be the 64-character SHA256 published on the Red Hat product download page for the Ansible automation portal installer image.
- `redhat_offline_token` comes from [access.redhat.com/management/api](https://access.redhat.com/management/api). The playbook exchanges it for an access token, looks up the image by checksum, and downloads it to `download_dir`. The download is checked against `image_sha256`.
- The last task prints the saved path. Set `qcow2_path` in `vars/vars_self_serve.yml` to that file before deploying. The default path expects `ansible-automation-portal-2.3.1-x86_64.qcow2`.

```bash
ansible-playbook download_image.yml
```

### 2. `deploy_self_serve_vm.yml`

Two plays, both aimed at `selfservehost`.

**Register the KVM host.** Runs `subscription-manager register` with `rhsm_org` and `rhsm_activation_key`. An already-registered host is treated as success.

**Deploy the portal VM.**

1. Installs KVM and libvirt and starts `libvirtd`.
2. Adds `ansible_user` to the `libvirt` group.
3. Renders `templates/user-data.j2` and `templates/meta-data.j2` under `work_dir` (`~/portal-deploy` on the host).
4. Builds a cloud-init ISO labeled `cidata`.
5. Copies `qcow2_path` to `/var/lib/libvirt/images/<vm_name>.qcow2` and copies the ISO next to it.
6. Defines the guest with `virt-install --import` on the libvirt `default` network.
7. Waits until the guest has an address on that network.
8. Adds iptables rules that forward TCP 443 on `portal_forward_iface` to the guest. These rules are not saved and do not survive a reboot of the KVM host.

The cloud-init user-data creates `portal_user` with passwordless sudo and installs `ssh_public_key`. It also writes the controller URL, API token, `aap_check_ssl`, OAuth client, database type, and portal base URL. That is the only place an SSH key is added, and it is the public key from the vault file.

```bash
ansible-playbook deploy_self_serve_vm.yml
```

When it finishes, open `https://<kvm-host>/`.

### 3. `destroy_self_server_vm.yml`

Stops the guest, undefines it, and deletes the cloud-init ISO in `work_dir` and under `/var/lib/libvirt/images`. Undefine uses `force: true`, which also deletes libvirt storage volumes attached to the guest. That removes the copied qcow2 in `/var/lib/libvirt/images`. It does not remove the downloaded original at `qcow2_path`.

The filename is `destroy_self_server_vm.yml`.

```bash
ansible-playbook destroy_self_server_vm.yml
```
