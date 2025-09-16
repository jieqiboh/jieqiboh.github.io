---
title: "automating stuff with ansible"
date: 2025-09-14
description: "-"
---

**Problem:** I often want to quickly set up my development environment on new machines, be it on freshly spun-up docker containers, or on newly-built PC running Ubuntu.

Some things I always want installed include: Python, C++, Git, neovim.
Shell scripting is often tedious and error prone, so I decided to try out Ansible.
While Ansible is often used for managing fleets of remote servers, I’m using it here to automate local setup and to configure running Docker containers the same way.

## What is Ansible? ##
Ansible is an automation tool that applies declarative instructions (YAML “playbooks”) to machines. You describe the desired state (e.g., packages installed), and Ansible converges the target to that state idempotently: running the same playbook twice should not change anything the second time.

## Files ##

### dev_basics.yml ###
This is the playbook, which is a list of tasks describing the desired state. (What to do)  

It comprises:
- a bootstrap step using raw to ensure Python exists (needed for modules),
- apt tasks to install packages,
- a small version report at the end.
```yaml
---
- name: Install basic dev tools (Neovim, Go, Git, Python) via apt
  hosts: all
  become: yes
  gather_facts: no

  vars:
    dev_packages:
      - neovim             # editor
      - golang             # go 
      - git                # version control
      - python3            # python interpreter
      - python3-pip        # python package manager
      - python3-venv       # stdlib venv module
      - build-essential    # gcc/ld/headers 
      - ca-certificates    # HTTPS certs for curl, git, etc.
      - curl               # downloader/HTTP client
      - unzip              # common utility

  pre_tasks:
    - name: Ensure target is Debian/Ubuntu (apt-based)
      raw: test -x /usr/bin/apt-get
      register: has_apt
      changed_when: false
      failed_when: has_apt.rc != 0 # check return code is 0 (true)

    - name: Ensure Python is present (so Ansible modules can run in bare containers)
      raw: |
        test -e /usr/bin/python3 || (apt-get update -y && apt-get install -y python3)
      changed_when: false

    - name: Gather facts
      setup:

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600
      environment:
        DEBIAN_FRONTEND: noninteractive

    - name: Install developer packages
      ansible.builtin.apt:
        name: "{{ dev_packages }}"
        state: present
      environment:
        DEBIAN_FRONTEND: noninteractive

    # Optional: quick version report
    - name: Show versions
      ansible.builtin.shell: |
        set -o pipefail
        nvim --version 2>/dev/null | head -n1 || true
        go version 2>/dev/null || true
        git --version 2>/dev/null || true
        python3 --version 2>/dev/null || true
        pip3 --version 2>/dev/null || true
      args:
        executable: /bin/bash
      register: versions
      changed_when: false

    - name: Print versions
      ansible.builtin.debug:
        msg: "{{ versions.stdout_lines }}"
```

### inventory.ini file ###
The inventory file declares targets: (Where to do it)  
I define the following:
- localhost (your machine) with ansible_connection=local
- a running Docker container with ansible_connection=docker and ansible_become=false (no sudo inside containers).

```ini
[local]
localhost ansible_connection=local

# Example: a running Ubuntu-based container named "dev-ubuntu"
#   docker run -d --name dev-ubuntu ubuntu:22.04 sleep infinity
[devcontainers]
dev-ubuntu ansible_connection=docker ansible_become=false
```

### To run ###
```
ansible-playbook -i inventory.ini dev_basics.yml -l local
ansible-playbook -i inventory.ini dev_basics.yml -l devcontainers
```

## Final Thoughts ##
I'll probably use ansible more as I discover other use cases, but that'll be all for now 🙃