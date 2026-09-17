
# 1. Create an encrypted file (Ansible will prompt for a vault password)
ansible-vault create secrets.yml

YAML
db_password: "SuperSecretPassword123!"
api_token: "xyz987654321_token"

Save and quit: 

:wq ENTER

cat secrets.yml

ansible-vault view secrets.yml


**********************************

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

Save and quit: 

:wq ENTER

***************************************

ansible-vault encrypt_string 'SuperSecretPassword123!' --name 'db_password'

The result of this - copy/paste into:

cat > test_vault.yml << 'EOF'
---
- name: Test Inline Encrypted Variable
  hosts: webservers
  vars:
    # Embedded inline encrypted string
    db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          35643761626139303137623539313934393363613265386663666536323061343033313137613537
          6162376131366237336165336335376636336633396231320a633965326365626364613137623231
          64376133353433633562373135373537386364656233313330393237663664303935326636326532
          6364346230353662350a623237646363316333653232303536616331303438623236613137626231
          61373438343962626464636362646662393365386334303939396366336463373535

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


When it drops the ERROR (There was a vault format error: Vault format unhexlify error: Odd-length string)  

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

**********************************************










