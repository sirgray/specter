
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
