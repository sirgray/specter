
cat > setup_webserver.yml << 'EOF'

- name: Deploy Configurable Web Server
  hosts: webservers
  become: true

  # 1. DEFINE YOUR VARIABLES HERE
  vars:
    web_port: 80
    site_title: "Welcome to DevOps Class"
    app_dir: "/var/www/html"

  tasks:
    - name: Ensure Nginx is installed
      apt:
        name: nginx
        state: present
        update_cache: yes 

    - name: Create index.html using variables
      copy:
        # Reference variables with {{ double_braces }}
        content: "<h1>{{ site_title }}</h1><p>Running on port {{ web_port }}</p>"
        dest: "{{ app_dir }}/index.html"
        mode: '0644'

EOF


- name: Update custom Nginx config file
      copy:
        content: "server { listen {{ web_port }}; root {{ app_dir }}; }"
        dest: /etc/nginx/conf.d/custom.conf
      # 'notify' triggers the handler ONLY if this file is modified or created
      notify: Restart Nginx

  # HANDLERS SECTION (Runs at the very end of the play)
  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted

--> RUN


cat > facts_test.yml << 'EOF'
---
- name: Display System Information
  hosts: all
  tasks:
    - name: Print OS and Memory Details
      debug:
        msg: "Server {{ inventory_hostname }} runs {{ ansible_distribution }} {{ ansible_distribution_version }} with {{ ansible_memtotal_mb }} MB of RAM."

EOF

--> RUN


# Create the main directory and subdirectories
mkdir -p my-ansible-project/group_vars my-ansible-project/host_vars

# Navigate into the project folder
cd my-ansible-project

# Create root-level files
touch inventory.ini site.yml

# Create group variable files
touch group_vars/all.yml group_vars/webservers.yml

# Create host variable file
touch host_vars/web1.yml

cat > group_vars/all.yml << 'EOF'
environment_name: "Production"
admin_email: sysadmin@example.com
EOF

cat > group_vars/webservers.yml << 'EOF'
http_port: 80
max_clients: 200
EOF

cat > site.yml << 'EOF'
---
- name: Demonstrate Variable Scoping
  hosts: webservers
  tasks:
    - name: Print variables from files
      debug:
        msg: "Environment: {{ environment_name }} | Port: {{ http_port }} | Admin: {{ admin_email }}"
EOF

--> RUN (site.yml)


*********************************************


templates/index.html.j2

<!DOCTYPE html>
<html>
<head>
    <title>{{ site_title | default('Default Server Title') }}</title>
</head>
<body>
    <h1>Server Information Card</h1>
    <ul>
        <li><strong>Hostname:</strong> {{ ansible_hostname }}</li>
        <li><strong>OS:</strong> {{ ansible_distribution }} {{ ansible_distribution_version }}</li>
        <li><strong>IP Address:</strong> {{ ansible_default_ipv4.address }}</li>
        <li><strong>Total Memory:</strong> {{ ansible_memtotal_mb }} MB</li>
        <li><strong>Core Count:</strong> {{ ansible_processor_vcpus }}</li>
    </ul>

    <h3>Active Services:</h3>
    <ul>
    {% for service in active_services %}
        <li>{{ service }}</li>
    {% endfor %}
    </ul>
</body>
</html>


group_vars/webservers.yml

site_title: "Automated DevOps Node"
active_services:
  - Nginx Web Server
  - Firewall Security Module
  - System Telemetry Collector

deploy_custom_site.yml

---
- name: Deploy Dynamic System Info Website
  hosts: webservers
  become: true

  tasks:
    - name: Ensure web package is installed
      package:
        name: nginx
        state: present

    - name: Generate dynamic HTML landing page
      template:
        src: templates/index.html.j2
        dest: /var/www/html/index.html
        mode: '0644'

    - name: Ensure web server is running
      service:
        name: nginx
        state: started
        enabled: yes






