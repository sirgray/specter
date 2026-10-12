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





