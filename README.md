# ansible-kubernetes-configuration
Ansible Kubernetes Configuration
``` yaml
- name: Convert DHCP to Static IP dynamically (safe version)
  hosts: all
  become: true
  gather_facts: yes
  vars:
    dns: 1.1.1.1  # Change to your preferred DNS (e.g., 8.8.8.8)
```
name: → A descriptive title for this playbook run.
hosts: all → Run this play on all hosts in your Ansible inventory file.
become: true → Run commands as root (using sudo). Needed for editing /tc/netplan/.
gather_facts: yes → Tells Ansible to collect system information (like IP address, hostname, etc.).
This lets us use variables such as ansible_default_ipv4.address.
vars: → Defines play-level variables.
dns: 1.1.1.1 → A variable we’ll use later in the Netplan config.

``` yaml 
  tasks:
    - name: Print current network details
      debug:
        msg: |
          Current network details:
          Interface : {{ ansible_default_ipv4.interface }}
          IP Address: {{ ansible_default_ipv4.address }}/{{ ansible_default_ipv4.netmask }}
          Gateway   : {{ ansible_default_ipv4.gateway }}
          DNS       : {{ ansible_dns.nameservers | join(', ') }}
```
tasks: → The list of steps (actions) this playbook will run on each host.
debug: → Prints messages to the terminal for visibility.
The text inside {{ ... }} are Jinja2 variables (templating).
Example:
{{ ansible_default_ipv4.interface }} → auto-detected default network interface (e.g. eth0)
{{ ansible_dns.nameservers | join(', ') }} → joins multiple DNS addresses into a string.
Purpose: Just to show what IP, gateway, and DNS you’re currently using (for verification).
``` yaml
    - name: Find existing Netplan files
      ansible.builtin.find:
        paths: /etc/netplan
        patterns: "*.yaml"
      register: netplan_files
```
ansible.builtin.find → Searches for files that match certain patterns.
paths: → Directory to search in.
patterns: → Match only files ending with .yaml.
register: → Stores the result of this task in a variable (netplan_files).
You can later use it like {{ netplan_files.files }} to access the list of files.
Purpose: Find all existing Netplan configuration files (e.g. /etc/netplan/01-netcfg.yaml).
``` yaml
    - name: Backup existing Netplan files
      ansible.builtin.copy:
        src: "{{ item.path }}"
        dest: "{{ item.path }}.bak"
        remote_src: yes
      loop: "{{ netplan_files.files }}"
      when: netplan_files.matched > 0
```
copy: → Normally copies files from the controller → remote host.
remote_src: yes → Tells Ansible the source (src) is already on the remote host.
loop: → Runs this task once for each file in netplan_files.files.
For example: /etc/netplan/01-netcfg.yaml, /etc/netplan/50-cloud-init.yaml, etc.
when: → A condition — only run this task if netplan_files.matched > 0 (i.e., files exist).
Purpose: Make a .bak copy of every Netplan file before modifying anything.

``` yaml
    - name: Disable malformed or legacy Netplan configs
      ansible.builtin.shell: |
        for f in /etc/netplan/*.yaml; do
          if grep -q "addresses: -" "$f"; then
            mv "$f" "$f.disabled"
          fi
        done
      args:
        executable: /bin/bash
```
shell: → Runs shell commands directly on the remote host.
The script loops through /etc/netplan/*.yaml:
Uses grep to check if the file contains the bad pattern addresses: -
If found, renames the file (e.g. /etc/netplan/01-netcfg.yaml.disabled)
args.executable ensures it runs with /bin/bash.
Purpose: Automatically disable old or broken Netplan files that would cause YAML syntax errors.

``` yaml
    - name: Fix Netplan file permissions
      ansible.builtin.file:
        path: /etc/netplan
        mode: '0750'
        recurse: yes
```
file: → Used to manage permissions, ownership, and directories.
path: → Path to apply the changes.
mode: '0750' → Sets Unix permissions (owner = read/write/execute, group = read/execute, others = none).
recurse: yes → Apply this mode to all files inside /etc/netplan/.
Purpose: Fixes “Permissions too open” warnings from Netplan.

``` yaml 
 - name: Generate new Netplan configuration
      ansible.builtin.copy:
        dest: /etc/netplan/50-cloud-init.yaml
        content: |
          network:
            version: 2
            ethernets:
              {{ ansible_default_ipv4.interface }}:
                dhcp4: no
                addresses:
                  - {{ ansible_default_ipv4.address }}/{{ ansible_default_ipv4.prefix }}
                gateway4: {{ ansible_default_ipv4.gateway }}
                nameservers:
                  addresses:
                    - {{ dns }}
                    - 8.8.8.8
        mode: '0600'
``` 
copy: again — but this time we’re creating a new file (dest:).
content: → Inline content that will be written into the file.
Variables like {{ ansible_default_ipv4.address }} dynamically fill in host details.
mode: '0600' → File readable/writable only by root.
Purpose: Create a new Netplan YAML that sets a static IP instead of DHCP, using current system info.

``` yaml
    - name: Validate new Netplan configuration
      ansible.builtin.command:
        cmd: netplan generate
      register: netplan_generate
      failed_when: netplan_generate.rc != 0
      changed_when: false
```
command: → Runs a command (no shell expansion).
netplan generate → Checks and parses YAML files without applying them.
register: → Save result in variable netplan_generate.
failed_when: → Mark task as failed if return code (rc) is not 0.
changed_when: false → We’re just checking, so don’t mark it as a “change”.
Purpose: Validate Netplan syntax before applying, so we don’t break the network.

``` yaml
    - name: Apply Netplan configuration safely
      block:
        - name: Apply Netplan configuration
          ansible.builtin.command:
            cmd: netplan apply
          register: apply_result
          failed_when: apply_result.rc != 0
      rescue:
        - name: Rollback to previous configuration
          ansible.builtin.shell: |
            cp /etc/netplan/*.bak /etc/netplan/
            netplan apply
          ignore_errors: yes
        - name: Report failure and rollback
          ansible.builtin.debug:
            msg: "Netplan apply failed. Reverted to backup configuration."
```
This is a “block + rescue” structure — like try/catch in programming.
block: → Try running these tasks.
rescue: → If any task in the block fails, run these instead.
Inside the block:
Run netplan apply to apply new settings.
If it fails (rc != 0), jump to the rescue section.
Inside the rescue:
Copy back all .bak files to restore previous config.
Run netplan apply again to roll back.
Print a message that rollback was done.
Purpose: Apply configuration safely and automatically revert if something goes wrong.

Summary
Gathers info about current network.
Finds all Netplan YAML files.
Backs them up.
Disables old/broken YAMLs.
Fixes file permissions.
Writes a new static IP config.
Validates YAML.
Applies new config safely — rolls back if failed.