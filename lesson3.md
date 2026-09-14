
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








