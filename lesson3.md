
write a script loop_packages.yml to install multiple system packages in a single task:
Bash
cat > loop_packages.yml << 'EOF'
---
- name: Demonstrate Basic Loops
  hosts: webservers
  become: true

  tasks:
    # TASK 1: Refresh the repository cache
    - name: Update package cache
      package:
        update_cache: yes

    # TASK 2: Loop through and install packages
    - name: Install required baseline packages
      package:
        name: "{{ item }}"
        state: present
      loop:
        - curl
        - git
        - htop
        - unzip
EOF

**loop command is ALWAYS after the block**


cat > create_users.yml << 'EOF'
---
- name: User Management with Complex Loops
  hosts: webservers
  become: true

  vars:
    developers:
      - name: alice
        group: devops
      - name: bob
        group: developers
#two dimension array - 1-Alice 2-devops

  tasks:
    - name: Ensure target user groups exist
      group:
        name: "{{ item.group }}"
        state: present
      loop: "{{ developers }}"

    - name: Create developer accounts
      user:
        name: "{{ item.name }}"
        group: "{{ item.group }}"
        shell: /bin/bash
        state: present
      loop: "{{ developers }}"
EOF



cat > app_ports.yml << 'EOF'
---
- name: Loop Over Application Port Definitions
  hosts: webservers

  vars:
    application_ports:
      - name: http
        port: 80
      - name: https
        port: 443
      - name: app
        port: 8080

  tasks:
    - name: Display application port configuration
      ansible.builtin.debug:
        msg: "{{ item.name }} listens on port {{ item.port }}"
      loop: "{{ application_ports }}"
EOF

!! ansible.builtin - built-in ansible commands.


cat > loop_directories.yml << 'EOF'
---
- name: Create Multiple Application Directories
  hosts: webservers
  become: true

  tasks:
    - name: Create application directories
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        mode: "0755"
      loop:
        - /opt/myapp
        - /opt/myapp/config
        - /opt/myapp/logs
        - /opt/myapp/data
EOF


cat > multi_os_service.yml << 'EOF'
---
- name: Cross-Platform Web Server Setup
  hosts: webservers
  become: true

  tasks:
    - name: Install Apache on Debian/Ubuntu systems
      apt:
        name: apache2
        state: present
        update_cache: yes
      when: ansible_facts['os_family'] == "Debian"

    - name: Install Apache on RedHat/CentOS systems
      dnf:
        name: httpd
        state: present
      when: ansible_facts['os_family'] == "RedHat"

    - name: Ensure Apache is running (Debian)
      service:
        name: apache2
        state: started
      when: ansible_facts['os_family'] == "Debian"

    - name: Ensure Apache is running (RedHat)
      service:
        name: httpd
        state: started
      when: ansible_facts['os_family'] == "RedHat"
EOF

!! work with operators (==) 


Bash
cat > conditional_marker.yml << 'EOF'
---
- name: Conditional Configuration Using Registered Output
  hosts: webservers
  become: true

  tasks:
    - name: Check for application marker
      ansible.builtin.stat:
        path: /opt/myapp/installed.flag
      register: app_marker

    - name: Create application marker when missing
      ansible.builtin.file:
        path: /opt/myapp/installed.flag
        state: touch
        mode: "0644"
      when: not app_marker.stat.exists
EOF


Bash
cat > distro_packages.yml << 'EOF'
---
- name: Install Packages Based on Linux Distribution
  hosts: webservers
  become: true

  tasks:
    - name: Install nginx on Ubuntu
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
      when: ansible_facts["distribution"] == "Ubuntu"

    - name: Install nginx on Debian
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
      when: ansible_facts["distribution"] == "Debian"
EOF



Bash
cat > error_handling.yml << 'EOF'
---
- name: Demonstrate Advanced Error Control
  hosts: webservers
  tasks:

    # 1. Ignore errors for non-critical health checks
    - name: Ping an external host (Allowed to fail)
      command: ping -c 2 10.255.255.1
      ignore_errors: true

    # 2. Suppress false "changed" states for read-only commands
    - name: Check system uptime
      command: uptime
      changed_when: false    # Prevents Ansible from marking read-only checks as "changed"

    # 3. Define custom failure conditions based on command output
    - name: Check available disk space on root partition
      shell: df -h / | awk 'NR==2 {print $5}' | sed 's/%//'
      register: disk_usage
      changed_when: false
      failed_when: disk_usage.stdout | int > 90   # Fail ONLY if disk usage > 90%

    - name: Display disk check result
      debug:
        msg: "Root disk usage is currently at {{ disk_usage.stdout }}%"
EOF

**$$$ - create this yml to send alert (msg) ONLY disk usage < 35% and another alert (msg) if memory usage > 60%. $$$**





Bash
cat > read_only_check.yml << 'EOF'
---
- name: Read-Only System Check
  hosts: webservers

  tasks:
    - name: Check running kernel
      ansible.builtin.command: uname -r
      register: kernel_version
      changed_when: false

    - name: Display kernel version
      ansible.builtin.debug:
        msg: "Running kernel: {{ kernel_version.stdout }}"
EOF

!! This is to demonstrate changed_when


*******************************************************
Bash
cat > custom_failure.yml << 'EOF'
---
- name: Custom Failure Logic
  hosts: webservers

  tasks:
    - name: Check whether debug mode is enabled
      ansible.builtin.shell: "grep -q '^DEBUG=true' /etc/myapp/app.conf"
      register: debug_check
      changed_when: false
      failed_when: debug_check.rc not in [0, 1]

    - name: Display debug mode status
      ansible.builtin.debug:
        msg: >-
          Debug mode is
          {% if debug_check.rc == 0 %}enabled{% else %}not enabled{% endif %}
EOF
Run the playbook:
ansible-playbook -i inventory.ini custom_failure.yml


ERROR FIX

cat > failed_when.yml << 'EOF'
---
- name: Demonstrate failed_when
  hosts: webservers
  become: true

  tasks:

    - name: Create application configuration
      ansible.builtin.file:
        path: /etc/myapp
        state: directory
        mode: "0755"

    - name: Create configuration file
      ansible.builtin.copy:
        dest: /etc/myapp/app.conf
        content: |
          APP_NAME=myapp
          DEBUG=false
        mode: "0644"

    - name: Check DEBUG configuration
      ansible.builtin.command:
        cmd: grep -q '^DEBUG=true' /etc/myapp/app.conf
      register: debug_check
      changed_when: false
      failed_when: debug_check.rc == 2

    - name: Show result
      ansible.builtin.debug:
        msg: "DEBUG=true is enabled"
      when: debug_check.rc == 0
EOF

ansible-playbook -i inventory.ini failed_when.yml


ansible-playbook -i inventory.ini custom_failure.yml



*******************************************************









