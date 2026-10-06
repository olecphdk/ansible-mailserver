# Testing

The repository uses a deliberately small test setup.

## GitHub Actions

Every push and pull request runs:

```bash
ansible-playbook playbooks/setup-mailserver.yml --syntax-check
ansible-lint --profile min
```

The workflow runs in a fresh GitHub-hosted runner after checking out the relevant commit. It does not connect to the production mail server and does not use production SSH keys or Ansible Vault secrets.

The purpose is to catch YAML/Ansible syntax errors and basic structural problems before changes are deployed.

## Local checks

Run the same checks on the Ansible controller when useful:

```bash
ansible-playbook playbooks/setup-mailserver.yml --syntax-check
ansible-lint --profile min
```

Production behavior such as SMTP, IMAP, TLS, DKIM and Fail2ban is still verified on the server after deployment. CI deliberately does not have access to the production host.
