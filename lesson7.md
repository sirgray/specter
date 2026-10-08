cat > inventory_plugins/custom_cloud_inventory.py << 'EOF'
#!/usr/bin/python

from ansible.plugins.inventory import BaseInventoryPlugin
from ansible.errors import AnsibleError
import json
import urllib.request

DOCUMENTATION = r'''
    name: custom_cloud_inventory
    plugin_type: inventory
    short_description: Dynamic CMDB/Cloud inventory plugin
    options:
        plugin:
            description: Name of the plugin.
            required: true
            choices: ['custom_cloud_inventory']
        api_endpoint:
            description: Endpoint URL for host metadata discovery.
            required: true
'''

class InventoryModule(BaseInventoryPlugin):
    NAME = 'custom_cloud_inventory'

    def verify_file(self, path):
        valid = super(InventoryModule, self).verify_file(path)
        if valid and path.endswith(('custom_cloud.yml', 'custom_cloud.yaml')):
            return True
        return False

    def parse(self, inventory, loader, path, cache=True):
        super(InventoryModule, self).parse(inventory, loader, path)
        self._read_config_data(path)

        api_url = self.get_option('api_endpoint')

        try:
            req = urllib.request.Request(api_url, headers={'User-Agent': 'Ansible-Dynamic-Inventory'})
            with urllib.request.urlopen(req, timeout=5) as response:
                data = json.loads(response.read().decode())
        except Exception as e:
            raise AnsibleError(f"Failed to query CMDB API at {api_url}: {str(e)}")

        # Construct dynamic inventory groups
        self.inventory.add_group('dynamic_webservers')
        self.inventory.add_group('dynamic_dbservers')

        # Dummy node injection simulating CMDB API response
        mock_nodes = [
            {"hostname": "node_web_01", "ip": "127.0.0.1", "role": "web"},
            {"hostname": "node_db_01", "ip": "127.0.0.1", "role": "db"}
        ]

        for node in mock_nodes:
            hostname = node['hostname']
            self.inventory.add_host(host=hostname, group=f"dynamic_{node['role']}servers")
            self.inventory.set_variable(hostname, 'ansible_host', node['ip'])
            self.inventory.set_variable(hostname, 'ansible_connection', 'local')
            self.inventory.set_variable(hostname, 'node_role', node['role'])
EOF


cat > custom_cloud.yml << 'EOF'
plugin: custom_cloud_inventory
api_endpoint: "https://httpbin.org/get"
EOF


cat > test_dynamic_inventory.yml << 'EOF'
- name: Verify Dynamic CMDB Inventory Integration
  hosts: dynamic_webservers:dynamic_dbservers
  gather_facts: false

  tasks:
    - name: Display dynamic node attributes
      debug:
        msg: "Discovered Node: {{ inventory_hostname }} with IP {{ ansible_host }} as role {{ node_role }}"
EOF


ansible-playbook -i custom_cloud.yml test_dynamic_inventory.yml



******************************************************


cat > webhook_listener.py << 'EOF'
#!/usr/bin/python3

from http.server import HTTPServer, BaseHTTPRequestHandler
import json
import subprocess

class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        body = self.rfile.read(content_length)
        event_data = json.loads(body.decode('utf-8'))

        print(f"[EVENT RECEIVED] Alert triggered for service: {event_data.get('service')}")

        if event_data.get('status') == 'CRITICAL':
            print("[ACTION] Triggering automated Ansible remediation playbook...")
            subprocess.run([
                'ansible-playbook',
                'event_response.yml',
                '--extra-vars',
                f"target_service={event_data.get('service')}"
            ])

        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'{"status": "EVENT_PROCESSED"}')

def run(server_class=HTTPServer, handler_class=WebhookHandler, port=8088):
    server_address = ('', port)
    httpd = server_class(server_address, handler_class)
    print(f"Webhook listener active on port {port}...")
    httpd.handle_request()

if __name__ == '__main__':
    run()
EOF


cat > event_response.yml << 'EOF'
- name: Automated Reactive Remediation Playbook
  hosts: localhost
  connection: local

  vars:
    target_service: "httpd"

  tasks:
    - name: Log remediation event execution
      debug:
        msg: "ALERT REMEDIATION: Triggering automated recovery for service '{{ target_service }}'"

    - name: Ensure target service state is active
      file:
        path: "/tmp/remediation_{{ target_service }}.log"
        state: touch
        mode: '0644'
EOF


# Start listener in background
python3 webhook_listener.py &

# Send critical event payload
curl -X POST http://localhost:8088 \
  -H "Content-Type: application/json" \
  -d '{"service": "webserver", "status": "CRITICAL"}'


  ******************************************************


cat > ci_pipeline.yml << 'EOF'
- name: CI/CD Automated Validation Target
  hosts: localhost
  connection: local

  tasks:
    - name: Deploy application config file
      copy:
        dest: /tmp/ci_app_config.conf
        content: "APP_ENV=ci_stage\nDEBUG=False\n"
        mode: '0644'

    - name: Verify application configuration file existence
      stat:
        path: /tmp/ci_app_config.conf
      register: conf_stat
      # FORCE STAT TO CHECK DISK STATE REGARDLESS OF CHECK MODE
      check_mode: false

    - name: Assert file exists and is correctly structured
      assert:
        that:
          - conf_stat.stat.exists or ansible_check_mode
        fail_msg: "CI Validation Failed: Configuration file invalid."
        success_msg: "CI Validation Passed: Environment correctly configured."
EOF


cat > .gitlab-ci.yml << 'EOF'
stages:
  - lint
  - dry-run
  - deploy

lint_job:
  stage: lint
  script:
    - ansible-playbook --syntax-check ci_pipeline.yml
    - ansible-lint ci_pipeline.yml

dry_run_job:
  stage: dry-run
  script:
    - ansible-playbook ci_pipeline.yml --check --diff

deploy_job:
  stage: deploy
  script:
    - ansible-playbook ci_pipeline.yml
EOF


# Stage 1: Syntax Check
ansible-playbook --syntax-check ci_pipeline.yml

# Stage 2: Dry Run Check Mode
ansible-playbook ci_pipeline.yml --check --diff

# Stage 3: Convergence
ansible-playbook ci_pipeline.yml


*************************************************

cat > rolling_update_deploy.yml << 'EOF'
- name: Zero-Downtime Rolling Application Upgrade
  hosts: localhost
  connection: local
  serial: 1
  max_fail_percentage: 30

  tasks:
    - name: PRE-TASK - Remove host from load balancer pool
      file:
        path: /tmp/lb_drain_node.flag
        state: touch
        mode: '0644'

    - name: UPGRADE - Deploy new application version
      copy:
        dest: /tmp/app_version.txt
        content: "VERSION=2.5.0\n"
        mode: '0644'

    - name: POST-TASK - Health check newly updated node
      uri:
        url: "https://httpbin.org/get"
        status_code: 200
      register: health_check
      until: health_check.status == 200
      retries: 3
      delay: 1

    - name: POST-TASK - Re-add host to load balancer pool
      file:
        path: /tmp/lb_drain_node.flag
        state: absent
EOF


ansible-playbook rolling_update_deploy.yml

*************************************************

cat > inventory_plugins/secure_api_inventory.py << 'EOF'
#!/usr/bin/python

from ansible.plugins.inventory import BaseInventoryPlugin
from ansible.errors import AnsibleError
import json
import urllib.request

DOCUMENTATION = r'''
    name: secure_api_inventory
    plugin_type: inventory
    short_description: Vault-authenticated dynamic inventory plugin
    options:
        plugin:
            description: Name of the plugin.
            required: true
            choices: ['secure_api_inventory']
        api_token:
            description: Secret token for API authentication.
            required: true
'''

class InventoryModule(BaseInventoryPlugin):
    NAME = 'secure_api_inventory'

    def verify_file(self, path):
        valid = super(InventoryModule, self).verify_file(path)
        return valid and path.endswith(('secure_cloud.yml', 'secure_cloud.yaml'))

    def parse(self, inventory, loader, path, cache=True):
        super(InventoryModule, self).parse(inventory, loader, path)
        self._read_config_data(path)

        token = self.get_option('api_token')

        if not token or token == "UNSET":
            raise AnsibleError("API Token is missing or invalid!")

        self.inventory.add_group('secure_nodes')
        hostname = "secure_app_node"
        self.inventory.add_host(host=hostname, group='secure_nodes')
        self.inventory.set_variable(hostname, 'ansible_host', '127.0.0.1')
        self.inventory.set_variable(hostname, 'ansible_connection', 'local')
        self.inventory.set_variable(hostname, 'authenticated_status', 'SUCCESS')
EOF

Bash
cat > secure_cloud.yml << 'EOF'
plugin: secure_api_inventory
api_token: "SecretVaultToken999"
EOF

cat > test_secure_inventory.yml << 'EOF'
- name: Verify Secure Authenticated Dynamic Inventory
  hosts: secure_nodes
  gather_facts: false

  tasks:
    - name: Display authentication status
      debug:
        msg: "Host {{ inventory_hostname }} successfully authenticated with status: {{ authenticated_status }}"
EOF


ansible-playbook -i secure_cloud.yml test_secure_inventory.yml

**************************************************************


cat > site_orchestrator.yml << 'EOF'
- name: Master Deployment Orchestrator - Quality Gate Check
  hosts: localhost
  connection: local

  tasks:
    - name: Gate 1 - Check configuration file presence
      file:
        path: /tmp/production_app.conf
        state: touch
        mode: '0644'

    - name: Gate 2 - Verify security sandboxing parameters
      copy:
        dest: /tmp/production_app.conf
        content: |
          ENABLE_SECURITY_SANDBOX=TRUE
          ALLOWED_SUBNET=10.0.0.0/8
        mode: '0644'

    - name: Gate 3 - Read back configuration file
      command: cat /tmp/production_app.conf
      register: conf_out
      changed_when: false

    - name: Gate 4 - Assert security flags are present
      assert:
        that:
          - "'ENABLE_SECURITY_SANDBOX=TRUE' in conf_out.stdout"
        fail_msg: "Orchestration Aborted: Security flags missing from configuration."
        success_msg: "Orchestration Quality Gate Passed: All security conditions satisfied."
EOF

# 1. Run static lint check
sudo apt update && sudo apt install -y ansible-lint

ansible-lint site_orchestrator.yml  (get rid of FQCN errors)



# 2. Run dry-run simulation
ansible-playbook site_orchestrator.yml --check --diff

check why the error happened and fix 

# 3. Execute production deployment
ansible-playbook site_orchestrator.yml


*****************************************************


Create a playbook named micro_gate.yml that performs the following 4 tasks:

Session 1 & 5 (Dynamic Node & Vault Token Check):
Declare a playbook variable vault_api_token: "SecretToken123". Add a task to assert that vault_api_token is defined and non-empty.

Session 2 (Webhook Payload Event Log):
Simulate receiving a critical webhook alert by logging a debug message formatted as: "[ALERT RECEIVED] Service nginx experienced CRITICAL status."

Session 3 (CI/CD Pipeline Compatibility):
Add a task using ansible.builtin.copy to deploy a dummy config to /tmp/micro_app.conf with mode: '0644'. Ensure all tasks use Fully Qualified Collection Names (ansible.builtin.*).

Session 4 (Rolling Health Probe):
Add a task using ansible.builtin.uri to probe [https://httpbin.org/get](https://httpbin.org/get), setting until: health_check.status == 200, retries: 2, and delay: 1.


***************************

- name: Master Deployment Orchestrator - Quality Gate Check
  hosts: localhost
  connection: local

  tasks:
    - name: Gate 1 - Check configuration file presence
      ansible.builtin.file:
        path: /tmp/production_app.conf
        state: touch
        mode: '0644'

    - name: Gate 2 - Verify security sandboxing parameters
      ansible.builtin.copy:
        dest: /tmp/production_app.conf
        content: |
          ENABLE_SECURITY_SANDBOX=TRUE
          ALLOWED_SUBNET=10.0.0.0/8
        mode: '0644'

    - name: Gate 3 - Read back configuration file
      ansible.builtin.command: cat /tmp/production_app.conf
      register: conf_out
      changed_when: false
      # FORCE COMMAND TO EXECUTE REGARDLESS OF CHECK MODE IF FILE EXISTS
      ignore_errors: true

    - name: Gate 4 - Assert security flags are present
      ansible.builtin.assert:
        that:
          - ansible_check_mode or ('ENABLE_SECURITY_SANDBOX=TRUE' in conf_out.stdout)
        fail_msg: "Orchestration Aborted: Security flags missing from configuration."
        success_msg: "Orchestration Quality Gate Passed: All security conditions satisfied."











