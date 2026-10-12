cat > action_plugins/cloud_validator.py << 'EOF'
#!/usr/bin/python

from ansible.plugins.action import ActionBase
import os

class ActionModule(ActionBase):
    ''' Custom Action Plugin for Local Cloud Pre-Flight Validation '''

    def run(self, tmp=None, task_vars=None):
        if task_vars is None:
            task_vars = dict()

        result = super(ActionModule, self).run(tmp, task_vars)
        del tmp  # Unused path placeholder

        # Extract arguments passed from the task
        required_env = self._task.args.get('required_env', 'PRODUCTION')
        
        # Local control-node environment validation
        cloud_token = os.environ.get('CLOUD_API_TOKEN', 'MOCK_TOKEN_999')

        if not cloud_token:
            result['failed'] = True
            result['msg'] = "Pre-flight Validation Failed: CLOUD_API_TOKEN is missing on control node."
            return result

        result['changed'] = False
        result['validated_env'] = required_env
        result['token_preview'] = f"{cloud_token[:4]}****"
        result['msg'] = "Control-node cloud environment successfully validated."

        return result
EOF


cat > test_action_plugin.yml << 'EOF'
- name: Verify Custom Cloud Action Plugin Operations
  hosts: localhost
  connection: local

  tasks:
    - name: Run local control-node pre-flight cloud check
      cloud_validator:
        required_env: "STAGING"
      register: action_out

    - name: Display action plugin output
      ansible.builtin.debug:
        msg: "Validation Status: {{ action_out.msg }} (Token: {{ action_out.token_preview }})"

    - name: Assert action plugin execution succeeded
      ansible.builtin.assert:
        that:
          - action_out.validated_env == "STAGING"
        fail_msg: "Action Plugin Verification Failed."
        success_msg: "Action Plugin Verification Passed: Control node validated successfully."
EOF


ansible-playbook -i "localhost," test_action_plugin.yml


************************************************


cat > cloud_provision.yml << 'EOF'
- name: Declarative Cloud Infrastructure Provisioning
  hosts: localhost
  connection: local

  vars:
    cloud_cluster_name: "k8s-prod-cluster"
    target_node_count: 2

  tasks:
    - name: Generate cloud network baseline manifest
      ansible.builtin.copy:
        dest: /tmp/cloud_network.json
        content: |
          {
            "vpc_name": "{{ cloud_cluster_name }}-vpc",
            "cidr_block": "10.100.0.0/16",
            "status": "AVAILABLE"
          }
        mode: '0644'

    - name: Read created cloud network manifest
      ansible.builtin.slurp:
        src: /tmp/cloud_network.json
      register: vpc_raw

    - name: Parse network configuration attributes
      ansible.builtin.set_fact:
        vpc_data: "{{ vpc_raw.content | b64decode | from_json }}"

    - name: Provision dynamic cluster node manifests
      ansible.builtin.copy:
        dest: "/tmp/cloud_node_{{ item }}.json"
        content: |
          {
            "node_id": "node-{{ item }}",
            "vpc": "{{ vpc_data.vpc_name }}",
            "ip_address": "10.100.1.{{ item + 10 }}",
            "status": "RUNNING"
          }
        mode: '0644'
      loop: "{{ range(1, target_node_count + 1) | list }}"

    - name: Assert all cloud nodes provisioned in VPC
      ansible.builtin.stat:
        path: "/tmp/cloud_node_{{ item }}.json"
      register: node_stats
      loop: "{{ range(1, target_node_count + 1) | list }}"

    - name: Validate node existence
      ansible.builtin.assert:
        that:
          - item.stat.exists == true
        fail_msg: "Cloud Provisioning Failed: Missing cluster node configuration."
        success_msg: "Cloud Provisioning Passed: All nodes actively running in VPC."
      loop: "{{ node_stats.results }}"
EOF


ansible-playbook -i "localhost," cloud_provision.yml


**********************************************

cat > k8s_workload_deploy.yml << 'EOF'
- name: Kubernetes Microservice & Workload Deployment
  hosts: localhost
  connection: local

  vars:
    app_namespace: "production-apps"
    app_version: "v1.4.0"

  tasks:
    - name: Stage 1 - Render Kubernetes Namespace manifest
      ansible.builtin.copy:
        dest: /tmp/k8s_namespace.yml
        content: |
          apiVersion: v1
          kind: Namespace
          metadata:
            name: {{ app_namespace }}
            labels:
              environment: production
        mode: '0644'

    - name: Stage 2 - Render Kubernetes Deployment manifest
      ansible.builtin.copy:
        dest: /tmp/k8s_deployment.yml
        content: |
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: web-api-deployment
            namespace: {{ app_namespace }}
          spec:
            replicas: 3
            selector:
              matchLabels:
                app: web-api
            template:
              metadata:
                labels:
                  app: web-api
                  version: "{{ app_version }}"
              spec:
                containers:
                - name: api-container
                  image: nginx:1.25-alpine
                  ports:
                  - containerPort: 80
        mode: '0644'

    - name: Stage 3 - Read back generated deployment manifest
      ansible.builtin.command: cat /tmp/k8s_deployment.yml
      register: k8s_manifest_out
      changed_when: false

    - name: Stage 4 - Verify manifest attributes
      ansible.builtin.assert:
        that:
          - "'namespace: production-apps' in k8s_manifest_out.stdout"
          - "'replicas: 3' in k8s_manifest_out.stdout"
        fail_msg: "K8s Manifest Generation Failed: Invalid specification parameters."
        success_msg: "K8s Manifest Generation Passed: Deployment specification validated."
EOF


ansible-playbook -i "localhost," k8s_workload_deploy.yml

cat /tmp/k8s_deployment.yml


***************************************************


cat > secret_rotation.yml << 'EOF'
- name: Zero-Downtime Kubernetes Secret Rotation
  hosts: localhost
  connection: local

  vars:
    db_password_raw: "SuperSecretDBPass2026!"

  tasks:
    - name: Encode database password to Base64
      ansible.builtin.set_fact:
        db_password_b64: "{{ db_password_raw | b64encode }}"

    - name: Deploy Kubernetes Secret manifest
      ansible.builtin.copy:
        dest: /tmp/k8s_secret.yml
        content: |
          apiVersion: v1
          kind: Secret
          metadata:
            name: db-credentials
            namespace: production-apps
          type: Opaque
          data:
            DB_PASSWORD: {{ db_password_b64 }}
        mode: '0600'

    - name: Audit generated secret file permissions
      ansible.builtin.stat:
        path: /tmp/k8s_secret.yml
      register: secret_stat

    - name: Assert secret file is locked to root read-write (0600)
      ansible.builtin.assert:
        that:
          - secret_stat.stat.mode == '0600'
        fail_msg: "SECURITY VIOLATION: Secret file permissions are too permissive!"
        success_msg: "SECURITY PASSED: Secret file correctly restricted to owner read-write."
EOF



ansible-playbook -i "localhost," secret_rotation.yml

***************************************

cat > library/k8s_pod_health.py << 'EOF'
#!/usr/bin/python

from ansible.module_utils.basic import AnsibleModule

def run_module():
    module_args = dict(
        pod_name=dict(type='str', required=True),
        running_phase=dict(type='str', default='Running'),
        restart_count=dict(type='int', default=0)
    )

    result = dict(
        changed=False,
        healthy=False,
        pod_name=""
    )

    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    pod_name = module.params['pod_name']
    phase = module.params['running_phase']
    restarts = module.params['restart_count']

    result['pod_name'] = pod_name

    # Evaluate pod health logic
    if phase != 'Running':
        module.fail_json(
            msg=f"Pod '{pod_name}' is unready! Current status phase is '{phase}'.",
            **result
        )

    if restarts > 5:
        module.fail_json(
            msg=f"Pod '{pod_name}' is unstable! Restart count ({restarts}) exceeds threshold.",
            **result
        )

    result['healthy'] = True
    module.exit_json(**result)

def main():
    run_module()

if __name__ == '__main__':
    main()
EOF


cat > test_pod_health.yml << 'EOF'
- name: Verify Custom Kubernetes Pod Health Module
  hosts: localhost
  connection: local

  tasks:
    - name: Run health check on active web API pod
      k8s_pod_health:
        pod_name: "web-api-pod-01"
        running_phase: "Running"
        restart_count: 1
      register: health_out

    - name: Display health result
      ansible.builtin.debug:
        msg: "Pod {{ health_out.pod_name }} Health State: {{ health_out.healthy }}"
EOF


ansible-playbook -i "localhost," test_pod_health.yml


*******************************************************



Task 1: Build a Custom Action Plugin to Audit Local Disk Space

•	Objective: Practice writing control-node action plugins using ActionBase.
•	Instructions:
1.	Inside action_plugins/, create disk_audit.py.
2.	Implement an action plugin that checks if the control node has at least 1GB of free space in /tmp.
3.	Write test_disk_action.yml to call your action plugin and output a debug message.


Task 2: Build a Kubernetes Ingress Manifest Template

•	Objective: Master declarative K8s manifest templating.
•	Instructions:
1.	Create a playbook deploy_ingress.yml with variables domain_name: "api.example.com" and service_port: 8080.
2.	Write a task that uses ansible.builtin.copy to deploy a Kubernetes Ingress YAML manifest to /tmp/k8s_ingress.yml.
3.	Use ansible.builtin.command: cat /tmp/k8s_ingress.yml and assert to verify api.example.com is present in the rendered file.















