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







