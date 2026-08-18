
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

