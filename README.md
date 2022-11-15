# Lab - PiHole Containers

## Host Requirements

- Podman
- FirewallD

The Ansible Playbook will install all the latest versions of these packages on the target hosts.

## Using this repository

First off, the configuration in this repo is mine which means it's not likely to be usable by you - and you don't have edit access to this repo more than likely.  So, you'll need to fork this repo and then clone your forked repo to your local machine.

1. [Fork this repo](https://github.com/kenmoini/lab-pihole/fork)
2. Clone your forked repo to your local machine - `git clone https://github.com/YOUR_USERNAME/lab-pihole.git`.
3. If you'll be running the Playbook manually with the CLI then set up your inventory file - see the [examples/inventory](examples/inventory) file for an example of how to do this.  If you plan on using this with Ansible Tower you can skip creating the inventory file.
4. Now create your site configuration variable file - you can find the example file in the [examples/site_config.yml](examples/site_config.yml) file.  You'll need to edit this file to match your inventory and the services running on each host.  Write your customized version to `vars/site_config.yml`.
5. Now you can run the Playbook to deploy the PiHole services to your hosts.  If you're using the CLI, you can run the Playbook like this:

```bash=
ansible-playbook -i inventory deploy.yml
```

If you want to set per-service Web UI passwords, inspect the "Using Ansible Vault for secrets" section below.

> If you're using Ansible Tower or the AAP2 Controller, continue to the next step.

## Setting Up Ansible Tower for GitOps-y goodness

Ideally as soon as you commit a change to the `main` branch of this repo, the changes will be synced to the actual services running on the hosts.  To do this, we'll use Ansible Tower to run the `deploy.yml` playbook.  Assuming you already have Ansible Tower or the AAP2 Controller installed and ready to go, you can follow these steps to set things up:

1. Create a **Credential(s)** for the systems that you will be connecting to.  You can use the `Machine` credential type for this.  You can also use the `Source Control` credential type for the Git repo that you'll be using for the configuration files.
2. Create an **Inventory** for the hosts that you'll be deploying the DNS services to.
3. Add the **Hosts** to the **Inventory**.
4. Create a **Project** for your fork of this Git repo.  I like to keep the names the same as the repo, so I named mine `lab-pihole`.  Make sure to check the `Update Revision on Launch` box.
5. Create a **Template** for the `deploy.yml` Playbook, add the Credentials you'll need to connect to the hosts.  Important: Make sure to check the `Enable Webhook` box.  This will allow you to trigger the Playbook from a GitHub/GitLab webhook.
6. Once the Template is created, you can get the `Webhook URL` and the `Webhook Key` from the **Details** page of the Template.  You'll need these to set up the Webhook in GitHub/GitLab.

Assuming you'll be using this with GitHub, you can follow these steps to set up the Webhook: https://docs.ansible.com/ansible-tower/latest/html/userguide/webhooks.html

> With this, now when you perform a `git push` to the repo to update your PiHole configuration, the Playbook will be triggered and the changes will be automatically deployed to the hosts!

---

## Using Ansible Vault for secrets

You can store the passwords to the PiHole web interface in a file encrypted with Ansible Vault. This way you can keep the passwords in a git repository without having to worry about them being compromised.

### Create a vault file

The following structure stores the passwords for the services on the deployed hosts:

```yaml=
service_passwords:
  podman-host.example.com: # should match the inventory_hostname
    dns-pihole-1: "password" # k:v pair for the service.name: password
    dns-pihole-2: "password"
```

Ideally create it at `vars/vault.yml` because the Playbook will include it if it exists.

### Encrypt the vault file

Enter a password when prompted

```bash
# Create a new vaulted file from that structure
ansible-vault encrypt vars/vault.yml
```

### Use the vault file via the Ansible CLI

The Playbook will automatically include the vault file if it exists, but you will still need to provide it the password to decrypt it.

```bash
ansible-playbook -e "@./vars/vault.yml" -i inventory --ask-vault-pass deploy.yml
```

### Using the vault file with Ansible Tower

If you're using Ansible Tower, you can create a Credential of type `Vault` and use that to provide the password to decrypt the vault file when configuring the Template or running the Job.
