Ansible automates service mesh configurations by declaratively rendering 
PeerAuthentication (mTLS enforcing) and VirtualService (traffic management) 
manifests to enforce strict zero-trust network policies.



cat > connection_plugins/mesh_exec.py << 'EOF'
#!/usr/bin/python

from ansible.plugins.connection import ConnectionBase
import subprocess

class Connection(ConnectionBase):
    ''' Custom Connection Plugin for Mesh Command Wrappers '''

    transport = 'mesh_exec'
    has_pipelining = False

    def _connect(self):
        self._connected = True
        return self

    def exec_command(self, cmd, in_data=None, sudoable=True):
        super(Connection, self).exec_command(cmd, in_data=-1, sudoable=sudoable)
        
        # Execute commands directly on local control-node shell wrapper
        p = subprocess.Popen(
            cmd,
            shell=True,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        )
        stdout, stderr = p.communicate()
        return (p.returncode, stdout, stderr)

    def put_file(self, in_path, out_path):
        super(Connection, self).put_file(in_path, out_path)
        with open(in_path, 'rb') as f_in:
            with open(out_path, 'wb') as f_out:
                f_out.write(f_in.read())

    def fetch_file(self, in_path, out_path):
        super(Connection, self).fetch_file(in_path, out_path)
        self.put_file(in_path, out_path)

    def close(self):
        self._connected = False
EOF



cat > test_connection_plugin.yml << 'EOF'
- name: Verify Custom Connection Plugin Operations
  hosts: localhost
  connection: mesh_exec

  tasks:
    - name: Run command over custom transport wrapper
      ansible.builtin.command: echo "MESH_TRANSPORT_ACTIVE"
      register: transport_out

    - name: Display connection output
      ansible.builtin.debug:
        msg: "Transport Response: {{ transport_out.stdout }}"

    - name: Assert transport executed successfully
      ansible.builtin.assert:
        that:
          - "'MESH_TRANSPORT_ACTIVE' in transport_out.stdout"
        fail_msg: "Connection Plugin Verification Failed."
        success_msg: "Connection Plugin Verification Passed: Custom transport functional."
EOF

ansible-playbook -i "localhost," test_connection_plugin.yml



**********************************************************



cat > mesh_policy_deploy.yml << 'EOF'
- name: Service Mesh mTLS and VirtualService Policy Deployment
  hosts: localhost
  connection: local

  vars:
    mesh_namespace: "mesh-apps"
    mtls_mode: "STRICT"

  tasks:
    - name: Stage 1 - Render PeerAuthentication mTLS policy manifest
      ansible.builtin.copy:
        dest: /tmp/mesh_mtls_policy.yml
        content: |
          apiVersion: security.istio.io/v1beta1
          kind: PeerAuthentication
          metadata:
            name: default-mtls-policy
            namespace: {{ mesh_namespace }}
          spec:
            mtls:
              mode: {{ mtls_mode }}
        mode: '0644'

    - name: Stage 2 - Render VirtualService traffic routing manifest
      ansible.builtin.copy:
        dest: /tmp/mesh_virtualservice.yml
        content: |
          apiVersion: networking.istio.io/v1alpha3
          kind: VirtualService
          metadata:
            name: payment-service-router
            namespace: {{ mesh_namespace }}
          spec:
            hosts:
            - payment-service
            http:
            - route:
              - destination:
                  host: payment-service
                  subset: v1
                weight: 100
        mode: '0644'

    - name: Stage 3 - Read generated VirtualService policy file
      ansible.builtin.command: cat /tmp/mesh_virtualservice.yml
      register: vs_out
      changed_when: false

    - name: Stage 4 - Assert policy parameters
      ansible.builtin.assert:
        that:
          - "'weight: 100' in vs_out.stdout"
          - "'namespace: mesh-apps' in vs_out.stdout"
        fail_msg: "Mesh Policy Generation Failed: Incorrect policy definitions."
        success_msg: "Mesh Policy Generation Passed: VirtualService manifest validated."
EOF

ansible-playbook -i "localhost," mesh_policy_deploy.yml


**************************************


cat > ebpf_probe_deploy.yml << 'EOF'
- name: eBPF Kernel Telemetry Probe Deployment
  hosts: localhost
  connection: local

  vars:
    target_interface: "eth0"
    probe_sampling_rate: 1000

  tasks:
    - name: Stage 1 - Generate eBPF socket monitoring spec
      ansible.builtin.copy:
        dest: /tmp/ebpf_probe_config.json
        content: |
          {
            "interface": "{{ target_interface }}",
            "sampling_rate_hz": {{ probe_sampling_rate }},
            "trace_points": [
              "sys_enter_connect",
              "sys_exit_connect"
            ],
            "status": "ATTACHED"
          }
        mode: '0644'

    - name: Stage 2 - Read active probe spec
      ansible.builtin.slurp:
        src: /tmp/ebpf_probe_config.json
      register: raw_probe_data

    - name: Stage 3 - Extract eBPF probe variables
      ansible.builtin.set_fact:
        probe_spec: "{{ raw_probe_data.content | b64decode | from_json }}"

    - name: Stage 4 - Assert probe attached to kernel interface
      ansible.builtin.assert:
        that:
          - probe_spec.interface == "eth0"
          - probe_spec.status == "ATTACHED"
        fail_msg: "eBPF Probe Failure: Kernel telemetry probe not attached."
        success_msg: "eBPF Probe Passed: Kernel socket probes active on eth0."
EOF

ansible-playbook -i "localhost," ebpf_probe_deploy.yml

********************************************

cat > canary_traffic_shift.yml << 'EOF'
- name: Automated Canary Weight Shifting & Circuit Breaking
  hosts: localhost
  connection: local

  vars:
    v1_weight: 80
    v2_weight: 20

  tasks:
    - name: Apply 80/20 canary traffic split configuration
      ansible.builtin.copy:
        dest: /tmp/canary_route.json
        content: |
          {
            "service": "checkout-api",
            "subsets": {
              "v1": {{ v1_weight }},
              "v2": {{ v2_weight }}
            },
            "circuit_breaker": {
              "consecutive_5xx_errors": 3,
              "ejection_time_seconds": 30
            }
          }
        mode: '0644'

    - name: Read active canary routing state
      ansible.builtin.slurp:
        src: /tmp/canary_route.json
      register: canary_raw

    - name: Parse active canary configuration
      ansible.builtin.set_fact:
        canary_data: "{{ canary_raw.content | b64decode | from_json }}"

    - name: Assert sum of canary weights equals 100 percent
      ansible.builtin.assert:
        that:
          - (canary_data.subsets.v1 + canary_data.subsets.v2) == 100
        fail_msg: "Canary Rollout Aborted: Traffic routing weights do not sum to 100%!"
        success_msg: "Canary Rollout Passed: Valid 80/20 traffic weight allocation."
EOF


ansible-playbook -i "localhost," canary_traffic_shift.yml

******************************************************


Task 1: Build a Custom Connection Plugin Execution Counter

•	Objective: Practice extending ConnectionBase.
•	Instructions:
1.	Create connection_plugins/counter_exec.py.
2.	Modify exec_command to print "COMMAND_COUNT_LOGGED" to standard error before executing commands.
3.	Write test_counter.yml using connection: counter_exec to verify output.




Task 2: Build a Strict Mesh AccessPolicy Manifest

•	Objective: Practice declarative mesh policy generation.
•	Instructions:
1.	Create a playbook mesh_access_policy.yml.
2.	Render a YAML file to /tmp/mesh_authz.yml defining an Istio AuthorizationPolicy that allows HTTP GET requests only to path /healthz.
3.	Assert using ansible.builtin.command and assert that path: /healthz is present in the rendered file.









