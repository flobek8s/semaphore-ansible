# ansible-repo

Playbooks and inventory for the homelab. Run from Semaphore UI, or locally
for testing.

## Layout

```
ansible.cfg
requirements.yml          # ansible-galaxy collections
inventory/
  hosts.yml                          # plaintext, non-sensitive
  group_vars/all/vars.yml            # plaintext, non-sensitive
  group_vars/all/vault.yml           # ansible-vault ENCRYPTED, created by you, see below
  group_vars/all/vault.yml.example   # shows the shape, not real data
playbooks/
  site.yml              # imports everything
  update-ubuntu.yml      # apt update/upgrade/autoremove + conditional reboot
  vault-test.yml         # confirms vault decryption works end to end
  k8s-node-status.yml    # kubernetes.core demo (no kubectl binary needed)
  talos-status.yml       # talosctl binary demo
```

## Setting up Ansible Vault (local + Semaphore)

1. Generate a vault password and save it somewhere safe (a password
   manager, not this repo):
   ```bash
   openssl rand -base64 32 > ~/.ansible/vault_pass.txt
   chmod 600 ~/.ansible/vault_pass.txt
   ```
2. Create the real, encrypted vars file:
   ```bash
   ansible-vault create inventory/group_vars/all/vault.yml \
     --vault-password-file ~/.ansible/vault_pass.txt
   ```
   Put real secrets in using the shape shown in `vault.yml.example`.
3. Commit `inventory/group_vars/all/vault.yml` -- it's fine to commit, it's
   encrypted at rest. Never commit `vault_pass.txt` (already gitignored).
4. To edit later: `ansible-vault edit inventory/group_vars/all/vault.yml --vault-password-file ~/.ansible/vault_pass.txt`
5. To test locally:
   ```bash
   ansible-playbook playbooks/vault-test.yml --vault-password-file ~/.ansible/vault_pass.txt
   ```

## Wiring Vault into Semaphore

Semaphore does **not** read `~/.ansible/vault_pass.txt` or any file path --
it stores the vault password itself, encrypted in its own database, and
hands it to `ansible-playbook --vault-password-file` internally at run time.

1. In Semaphore: **Key Store → New Key**
   - Type: `Login with Password`
   - Name: `vault-password`
   - Login: leave blank
   - Password: the same value you put in `~/.ansible/vault_pass.txt`
2. On the **Task Template** for any playbook that needs vault vars, set
   **Vault Password** to the `vault-password` key above. You can attach
   more than one vault password to a single template if you ever split
   secrets across multiple vault files.
3. Run the `vault-test.yml` template first and confirm the output says
   `VAULT OK`.

## SSH access to managed hosts

Semaphore needs its own SSH key to reach `ubuntu-vm-01` etc. -- this is
separate from the Ansible Vault password above.

1. Generate a dedicated key (don't reuse your personal one):
   ```bash
   ssh-keygen -t ed25519 -C "semaphore-automation" -f ./semaphore_ansible_key -N ""
   ```
2. Put the **public** key in `authorized_keys` for the `ansible` user on
   each managed host (manually, or via cloud-init/your existing
   provisioning process for new VMs).
3. In Semaphore: **Key Store → New Key**
   - Type: `SSH Key`
   - Name: `ansible-ssh-key`
   - Paste the **private** key content
4. Attach `ansible-ssh-key` as the Access Key on the Inventory (or
   Repository, if it also needs to clone over SSH).
5. Delete the local private key file once it's pasted into Semaphore, or
   move it to your password manager. See `semaphore-deploy/secrets-templates/`
   if you also want a Sealed Secret copy committed to git for disaster
   recovery (recommended -- losing this key means re-provisioning trust on
   every managed host).

## Collections

```bash
ansible-galaxy collection install -r requirements.yml
```
This already happens automatically inside the custom Semaphore image (see
`semaphore-deploy/Dockerfile`), but run it locally too if you're testing
playbooks from your own machine.
