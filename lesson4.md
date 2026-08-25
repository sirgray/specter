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


*********************************************************************************

Bash
# 1. Create separate key files
echo "DevKey123" > .vault_pass_dev
echo "ProdKey999" > .vault_pass_prod

# 2. To prevent Ansible from trying to decrypt the old file with your new keys, remove it or rename its extension:

rm -f group_vars/webservers/vault.yml


#3 Re-create your encrypted variable using the dev vault identity so Ansible knows which key in your command line decrypts it:

# 1.1. Create a variable encrypted specifically with the 'dev' vault ID
ansible-vault encrypt_string --vault-id dev@.vault_pass_dev 'DevSecretPass123' --name 'dev_secret' > /tmp/dev_var.txt

# 1.2. Rebuild test_vault.yml with the newly labeled string
cat > test_vault.yml << 'EOF'
---
- name: Test Multi-Vault Execution
  hosts: webservers
  vars:
EOF

# Append the indented variable block
sed 's/^/    /' /tmp/dev_var.txt >> test_vault.yml

# Append the task block
cat >> test_vault.yml << 'EOF'

  tasks:
    - name: Print decrypted secret
      debug:
        msg: "Decrypted secret is: {{ dev_secret }}"
EOF

# 4. Run playbooks passing multiple vault IDs automatically
ansible-playbook -i inventory.ini test_vault.yml \
  --vault-id dev@.vault_pass_dev \
  --vault-id prod@.vault_pass_prod
 ************************************************************

 Bash
# Create vault password
echo "SuperClassSecret2026" > .vault_pass
chmod 600 .vault_pass

# Create group_vars directory
mkdir -p group_vars/webservers
Create group_vars/webservers/vault.yml using ansible-vault:
Bash
ansible-vault create group_vars/webservers/vault.yml --vault-password-file .vault_pass
Add the following data:
YAML
vault_db_name: "production_db"
vault_db_user: "admin_user"
vault_db_pass: "s3cur3_P@ssw0rd_2026!"
2. Create Application Config Template (10 min)
Have students write templates/db_config.php.j2:
Code snippet

mkdir -p templates

cat >> templates/db_config.php.j2 << 'EOF'
<?php
// Generated automatically by Ansible - DO NOT EDIT MANUALLY
define('DB_NAME', '{{ vault_db_name }}');
define('DB_USER', '{{ vault_db_user }}');
define('DB_PASSWORD', '{{ vault_db_pass }}');
define('DB_HOST', 'localhost');
?>
EOF


cat > deploy_secure_app.yml << 'EOF'
---
- name: Deploy Secure Database Configuration
  hosts: webservers
  become: true

  tasks:
    - name: Ensure target config directory exists
      file:
        path: /var/www/app
        state: directory
        mode: '0755'

    - name: Deploy database config with restricted permissions
      template:
        src: templates/db_config.php.j2
        dest: /var/www/app/db_config.php
        mode: '0600'   # Restrict read permissions to root only
      no_log: true     # PREVENTS SENSITIVE DATA FROM PRINTING IN TERMINAL LOGS
EOF


ansible-playbook -i inventory.ini deploy_secure_app.yml --vault-password-file .vault_pass


******************************************
Bash
# Step 1: Create a setup with an old password file
echo -n "OldPassword2026" > .vault_pass_old
echo -n "NewPassword2027" > .vault_pass_new
chmod 600 .vault_pass_old .vault_pass_new

# Step 2: Create a vaulted file with the old password

ansible-vault create group_vars/webservers/secrets.yml \
  --vault-password-file .vault_pass_old \
  --encrypt-vault-id default

  
app_db_pass: "SuperConfidentialPass123"


Bash
ansible-vault rekey group_vars/webservers/secrets.yml \
  --vault-password-file .vault_pass_old \
  --new-vault-password-file .vault_pass_new



Bash
cat > test_no_log.yml << 'EOF'
---
- name: Demonstrate Output Masking
  hosts: webservers
  vars_files:
    - group_vars/webservers/secrets.yml

  tasks:
    - name: Unsafe Task - Prints raw password to terminal logs
      debug:
        msg: "Exposed secret: {{ app_db_pass }}"

    - name: Safe Task - Sensitive step with output protection
      debug:
        msg: "Exposed secret: {{ app_db_pass }}"
      no_log: true   # Suppresses output printing in standard logs
EOF


ansible-playbook -i inventory.ini test_no_log.yml --vault-password-file .vault_pass_new (HANDLE the Error)








