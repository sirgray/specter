


1. Pick an environment with Docker preinstalled
Go to killercoda.com/docker and start any playground/scenario there (Docker is preinstalled). This single node becomes your controller.

2. Install Ansible on the controller
bash
apt update && apt install -y ansible

3. Create the 2 managed nodes as containers
bash
docker rm -f node1 node2
docker run -d --name node1 --privileged --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  geerlingguy/docker-ubuntu2004-ansible:latest
docker run -d --name node2 --privileged --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  geerlingguy/docker-ubuntu2004-ansible:latest
  
4. Write the inventory (inventory.ini), using the docker connection plugin so no SSH setup is needed:
cat > inventory.ini << 'EOF'
[webservers]
node1 ansible_connection=docker
node2 ansible_connection=docker
EOF

5. Verify connectivity
bash
ansible -i inventory.ini webservers -m ping

6. Run your playbook
(CREATE PLAYBOOK FIRST)

cat > playbook.yml << 'EOF'
---
- name: Deploy and Configure Nginx Web Server
  hosts: webservers
  become: true
  tasks:
    - name: Ensure Nginx is installed
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Create custom index.html page
      copy:
        content: "<h1>Welcome to Ansible Class! Managed by Playbook.</h1>"
        dest: /var/www/html/index.html
        mode: '0644'

    - name: Ensure Nginx service is started and enabled at boot
      service:
        name: nginx
        state: started
        enabled: yes
EOF
bash
ansible-playbook -i inventory.ini playbook.yml


