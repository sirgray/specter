apt update && apt install -y ansible



cat > inventory.ini << 'EOF'
[webservers]
node1 ansible_connection=docker
node2 ansible_connection=docker
EOF





docker rm -f node1 node2

docker run -d --name node1 --privileged --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  geerlingguy/docker-ubuntu2004-ansible:latest

docker run -d --name node2 --privileged --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  geerlingguy/docker-ubuntu2004-ansible:latest




***************************
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


**************************

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


      *****************************


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
****************************************

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


      *******************************

      ---
- name: Fault-Tolerant Web Deployment with Rollback
  hosts: webservers
  become: true

  vars:
    staging_dir: "/tmp/web_staging"
    app_dir: "/var/www/html"

  tasks:
    - name: Main Deployment Block (Try)
      block:
        - name: Create temporary staging directory
          file:
            path: "{{ staging_dir }}"
            state: directory

        - name: Staging deployment artifact
          copy:
            content: "<h1>Version 2.0 - Production Ready</h1>"
            dest: "{{ staging_dir }}/index.html"

        # SIMULATE FAILURE: Intentionally copying to an invalid directory to trigger rescue
        - name: Deploy artifact to application directory
          copy:
            src: "{{ staging_dir }}/index.html"
            dest: "{{ app_dir }}/index.html"
            remote_src: true

        - name: Confirm successful deployment
          debug:
            msg: "Deployment succeeded without issues!"

      rescue:
        - name: CRITICAL ERROR - Triggering Automated Rollback
          debug:
            msg: "Deployment failed! Rolling back to emergency maintenance page..."

        - name: Deploy emergency maintenance landing page
          copy:
            content: "<h1 style='color:red;'>System Under Emergency Maintenance</h1>"
            dest: "{{ app_dir }}/index.html"

      always:
        - name: CLEANUP - Remove temporary staging directory
          file:
            path: "{{ staging_dir }}"
            state: absent

        - name: Log execution timestamp
          debug:
            msg: "Deployment workflow finished execution at {{ ansible_date_time.iso8601 }}"




