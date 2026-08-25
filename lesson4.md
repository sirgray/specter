ansible-vault create secrets.yml

db_password: "SuperSecretPassword123!"
api_token: "xyz987654321_token"


:wq ENTER

ansible-vault view secrets.yml

mkdir -p group_vars/webservers

# Unencrypted variables
cat > group_vars/webservers/vars.yml << 'EOF'
db_host: "10.0.0.5"
db_port: 3306
db_user: "app_user"
EOF

# Encrypted secrets file created directly via CLI
ansible-vault create group_vars/webservers/vault.yml

vault_db_password: "ProductionPassword99!"

:wq ENTER


ansible-vault encrypt_string 'SuperSecretPassword123!' --name 'db_password'


cat > test_vault.yml << 'EOF'
---
- name: Test Inline Encrypted Variable
  hosts: webservers
  vars:
    # Embedded inline encrypted string
    db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          3663363332303233300a333934336132336364373462613565313833326164343163353733303632
          34313532323033333333306132333331393630653335313364383331613932356562303339343063
          3836373839353137633639323336386337323863363836330a333735393033303237303337373836

  tasks:
    - name: Display unencrypted secret during runtime
      debug:
        msg: "The decrypted password is: {{ db_password }}"
EOF


# 1. Store the vault password in a hidden local file
echo "123" > .vault_pass

# 2. Lock down permissions so only your user can read it
chmod 600 .vault_pass

# 3. CRITICAL: Add the password file to .gitignore so it NEVER gets committed!
echo ".vault_pass" >> .gitignore

ansible-playbook -i inventory.ini test_vault.yml --vault-password-file .vault_pass


FIXING:
1. 
ansible-vault encrypt_string 'YourSecret123!' --name 'db_password' --vault-password-file .vault_pass


2. 

# 1. Generate the encrypted variable block directly into a temporary file
ansible-vault encrypt_string 'YourSecret123!' --name 'db_password' --vault-password-file .vault_pass > /tmp/encrypted_var.txt

# 2. Build the new playbook
cat > test_vault.yml << 'EOF'
---
- name: Test Inline Encrypted Variable
  hosts: webservers
  vars:
EOF

# 3. Append the formatted vault block (indented correctly)
sed 's/^/    /' /tmp/encrypted_var.txt >> test_vault.yml

# 4. Append the task section
cat >> test_vault.yml << 'EOF'

  tasks:
    - name: Display unencrypted secret during runtime
      debug:
        msg: "The decrypted password is: {{ db_password }}"
EOF

3. 
ansible-playbook -i inventory.ini test_vault.yml --vault-password-file .vault_pass




