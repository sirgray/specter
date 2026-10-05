cat > filter_plugins/custom_filters.py << 'EOF'
#!/usr/bin/python

def mask_sensitive_url(url_string):
    """
    Masks credentials embedded in standard URIs.
    Example: http://admin:secret@host:8080 -> http://admin:****@host:8080
    """
    import re
    return re.sub(r'(://[^:]+:)[^@]+(@)', r'\1****\2', url_string)

def extract_ip_by_subnet(ip_list, subnet_prefix):
    """
    Filters a list of IP addresses returning only those matching a target subnet prefix.
    """
    return [ip for ip in ip_list if ip.startswith(subnet_prefix)]

class FilterModule(object):
    def filters(self):
        return {
            'mask_url': mask_sensitive_url,
            'filter_subnet': extract_ip_by_subnet
        }
EOF



cat > test_custom_filters.yml << 'EOF'
- name: Verify Custom Jinja2 Filter Extensions
  hosts: localhost
  connection: local
  vars:
    raw_api_url: "https://admin:SecretPassword123@api.internal.net/v1/deploy"
    network_interfaces:
      - "192.168.1.50"
      - "10.0.4.12"
      - "192.168.1.105"
      - "172.16.0.1"

  tasks:
    - name: Display masked API endpoint in output logs
      debug:
        msg: "Target API Endpoint: {{ raw_api_url | mask_url }}"

    - name: Filter subnet interfaces for internal routing
      debug:
        msg: "Matched 192.168.1.x IPs: {{ network_interfaces | filter_subnet('192.168.1.') }}"
EOF


ansible-playbook test_custom_filters.yml





cat > library/system_health.py << 'EOF'
#!/usr/bin/python

from ansible.module_utils.basic import AnsibleModule
import shutil

def run_module():
    module_args = dict(
        min_disk_space_gb=dict(type='int', required=True),
        mount_point=dict(type='str', required=False, default='/')
    )

    result = dict(
        changed=False,
        free_gb=0,
        status='OK'
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    total, used, free = shutil.disk_usage(module.params['mount_point'])
    free_gb = free // (2**30)
    result['free_gb'] = free_gb

    if free_gb < module.params['min_disk_space_gb']:
        result['status'] = 'LOW_SPACE'
        module.fail_json(
            msg=f"Free disk space on {module.params['mount_point']} ({free_gb} GB) is below required minimum ({module.params['min_disk_space_gb']} GB).",
            **result
        )

    module.exit_json(**result)

def main():
    run_module()

if __name__ == '__main__':
    main()
EOF


cat > test_health_module.yml << 'EOF'
- name: Validate System Health Custom Module
  hosts: localhost
  connection: local

  tasks:
    - name: Assert root partition has at least 5 GB free space
      system_health:
        min_disk_space_gb: 5
        mount_point: "/"
      register: health_check

    - name: Output current system storage state
      debug:
        msg: "Check passed! Total free storage available on root: {{ health_check.free_gb }} GB"
EOF




cat > callback_plugins/json_audit.py << 'EOF'
import json
import time
from ansible.plugins.callback import CallbackBase

class CallbackModule(CallbackBase):
    CALLBACK_VERSION = 2.0
    CALLBACK_TYPE = 'notification'
    CALLBACK_NAME = 'json_audit'

    def __init__(self):
        super(CallbackModule, self).__init__()
        self.start_time = time.time()

    def v2_runner_on_failed(self, result, ignore_errors=False):
        host = result._host.get_name()
        task_name = result._task.get_name()
        error_msg = result._result.get('msg', 'Unknown Error')
        
        audit_entry = {
            "event": "TASK_FAILURE",
            "host": host,
            "task": task_name,
            "error": error_msg,
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
        }
        self._display.display(f"[AUDIT LOG] {json.dumps(audit_entry)}")

    def v2_playbook_on_stats(self, stats):
        duration = round(time.time() - self.start_time, 2)
        self._display.display(f"[AUDIT LOG] Playbook completed execution in {duration} seconds.")
EOF


cat > test_callback_plugin.yml << 'EOF'
- name: Intercept Execution Events via Custom Callback
  hosts: localhost
  connection: local

  tasks:
    - name: Run successful system operation
      command: echo "Ansible callback operational"

    - name: Simulate task failure to trigger audit callback
      command: ls /nonexistent_directory_for_audit_test
      ignore_errors: true
EOF


ANSIBLE_CALLBACKS_ENABLED=json_audit ansible-playbook test_callback_plugin.yml





cat > systemd_hardening.yml << 'EOF'
- name: Enterprise Systemd Service Hardening
  hosts: localhost
  connection: local
  become: true

  tasks:
    - name: Create isolated system user for application execution
      user:
        name: secure_app
        system: true
        shell: /sbin/nologin
        create_home: false

    - name: Deploy hardened systemd service unit
      copy:
        dest: /etc/systemd/system/hardened_app.service
        owner: root
        group: root
        mode: '0644'
        content: |
          [Unit]
          Description=Hardened Enterprise Application Service
          After=network.target

          [Service]
          Type=simple
          User=secure_app
          ExecStart=/usr/bin/python3 -m http.server 8085
          Restart=on-failure
          
          # Security Sandboxing Options
          ProtectSystem=strict
          ProtectHome=true
          NoNewPrivileges=true
          PrivateTmp=true
          CapabilityBoundingSet=CAP_NET_BIND_SERVICE

          [Install]
          WantedBy=multi-user.target

    - name: Reload systemd manager configuration
      systemd:
        daemon_reload: true

    - name: Ensure hardened service is enabled and started
      systemd:
        name: hardened_app
        state: started
        enabled: true
EOF



cat > ha_repo_sync.yml << 'EOF'
- name: Multi-Control Node Synchronization
  hosts: localhost
  connection: local

  vars:
    primary_repo_path: "/opt/ansible_control/repository"
    backup_repo_path: "/opt/ansible_control/sync_backup"

  tasks:
    - name: Prepare control node directory structures
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - "{{ primary_repo_path }}"
        - "{{ backup_repo_path }}"

    - name: Synchronize playbooks across local mirror paths
      synchronize:
        src: "{{ primary_repo_path }}/"
        dest: "{{ backup_repo_path }}/"
        archive: true
        delete: true
      delegate_to: localhost

    - name: Audit synchronized file integrity
      stat:
        path: "{{ backup_repo_path }}"
      register: backup_dir_stat

    - name: Report HA directory status
      debug:
        msg: "Backup Repository Mirror operational: {{ backup_dir_stat.stat.exists }}"
EOF


sudo ansible-playbook systemd_hardening.yml
ansible-playbook ha_repo_sync.yml





cat > lookup_plugins/api_secret.py << 'EOF'
#!/usr/bin/python

import json
import urllib.request
from ansible.plugins.lookup import LookupBase
from ansible.errors import AnsibleError

class LookupModule(LookupBase):
    def run(self, terms, variables=None, **kwargs):
        ret = []
        endpoint_url = terms[0] if terms else "https://httpbin.org/json"

        try:
            req = urllib.request.Request(
                endpoint_url, 
                headers={'User-Agent': 'Ansible-Custom-Lookup'}
            )
            with urllib.request.urlopen(req, timeout=5) as response:
                if response.status == 200:
                    data = json.loads(response.read().decode())
                    ret.append(data)
                else:
                    raise AnsibleError(f"HTTP Error {response.status} fetching secret.")
        except Exception as e:
            raise AnsibleError(f"Failed to execute API lookup against {endpoint_url}: {str(e)}")

        return ret
EOF




cat > test_api_lookup.yml << 'EOF'
- name: Retrieve External Secrets via Custom Lookup Plugin
  hosts: localhost
  connection: local

  vars:
    api_payload: "{{ lookup('api_secret', 'https://httpbin.org/json') }}"

  tasks:
    - name: Display dynamically retrieved lookup key
      debug:
        msg: "Retrieved external API author: {{ api_payload.slideshow.author }}"

    - name: Validate payload integrity
      assert:
        that:
          - api_payload.slideshow.title is defined
        fail_msg: "API lookup payload failed validation check."
        success_msg: "API lookup successfully fetched remote payload."
EOF




sudo apt update && sudo apt install -y ansible-lint


cat > .ansible-lint << 'EOF'
skip_list:
  - 'yaml[line-length]'
  - 'experimental'
warn_list:
  - 'command-instead-of-module'
EOF



cat > quality_check.yml << 'EOF'
- name: Automated Quality Gate Converge and Assertion Suite
  hosts: localhost
  connection: local
  become: true

  tasks:
    - name: Converge - Install application dependencies
      package:
        name: curl
        state: present

    - name: Verify - Check package presence using system command
      command: curl --version
      register: curl_check
      changed_when: false

    - name: Assert - Verify curl returns zero exit code
      assert:
        that:
          - curl_check.rc == 0
          - "'curl' in curl_check.stdout"
        fail_msg: "Role verification failed: curl is not functioning correctly."
        success_msg: "Role verification passed: infrastructure state converged successfully."
EOF


ansible-lint quality_check.yml

sudo ansible-playbook quality_check.yml







Task 1: Create a Custom Module to Audit Host Uptime
Objective: Understand how Ansible modules interact with AnsibleModule argument parsing and JSON exit codes.

Instructions:

Inside library/, create a custom Python module named uptime_check.py.

Accept a single integer parameter max_days (required).

Use Python's psutil.boot_time() or parse /proc/uptime to calculate the system's current uptime in days.

If system uptime exceeds max_days, trigger module.fail_json() stating the system needs a reboot; otherwise, call module.exit_json() returning uptime_days.

Write a test playbook test_uptime.yml that executes your module.






Task 2: Write an Audit Callback to Track Execution Time
Objective: Learn how Ansible event hooks trap playbook runtime statistics.

Instructions:

Inside callback_plugins/, create a plugin named timer_audit.py.

Override the v2_playbook_on_start and v2_playbook_on_stats event hooks.

When the playbook completes, calculate the total elapsed runtime in seconds and print [TIME AUDIT] Total Execution Time: X seconds.

Run any existing playbook with ANSIBLE_CALLBACKS_ENABLED=timer_audit to verify your output displays at the end of the run.














