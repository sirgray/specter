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










