
apt update && apt install -y ansible



cat > inventory.ini << 'EOF'
[webservers]
node1 ansible_connection=docker
node2 ansible_connection=docker
EOF
