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





