# Ansible - Configuration Management & Automation

## What is Ansible?
Ansible is an agentless automation platform for configuring systems, deploying software, and orchestrating complex IT tasks. Uses YAML playbooks for simple, readable automation.

---

## Real-Time Scenario 1: Rapid Server Provisioning

### Scenario Description
After Terraform creates 100 new servers in AWS, they need:
- OS security updates
- Docker installed
- Application deployed
- Monitoring agents configured
- Log collection setup

Manually doing this on 100 servers would take days. Ansible automates it in minutes.

### Problem Without Ansible
```bash
# Manual approach - repeat 100 times!
ssh admin@server1
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
# ... repeat for 100 servers
Time: 5-7 days
Errors: Human mistakes multiply
```

### Ansible Solution

```yaml
# inventory/aws_ec2.yml - Define target servers
---
all:
  children:
    web_servers:
      hosts:
        server[01:100]:  # servers 1-100
          ansible_host: "{{ inventory_hostname }}.ec2.amazonaws.com"
          ansible_user: ubuntu
          ansible_ssh_private_key_file: ~/.ssh/aws-key.pem
    
    database_servers:
      hosts:
        db[01:05]:
          ansible_host: "{{ inventory_hostname }}.db.internal"

  vars:
    ansible_python_interpreter: /usr/bin/python3
```

```yaml
# playbooks/provision.yml - Configuration playbook
---
- name: Provision AWS EC2 Instances
  hosts: web_servers
  become: yes  # Run with sudo
  gather_facts: yes
  
  vars:
    docker_version: "20.10"
    app_version: "1.2.0"
  
  tasks:
    - name: Update system packages
      apt:
        update_cache: yes
        cache_valid_time: 3600
      changed_when: false
    
    - name: Install security updates
      apt:
        upgrade: dist
    
    - name: Install required packages
      apt:
        name:
          - curl
          - wget
          - git
          - htop
          - net-tools
          - apt-transport-https
          - ca-certificates
        state: present
    
    - name: Add Docker GPG key
      shell: |
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
      changed_when: false
    
    - name: Add Docker repository
      shell: |
        echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
        apt-get update
    
    - name: Install Docker
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
        state: present
    
    - name: Start Docker service
      systemd:
        name: docker
        enabled: yes
        state: started
    
    - name: Add ubuntu user to docker group
      user:
        name: ubuntu
        groups: docker
        append: yes
    
    - name: Pull application image
      docker_image:
        name: "myapp:{{ app_version }}"
        source: registry
    
    - name: Run application container
      docker_container:
        name: myapp
        image: "myapp:{{ app_version }}"
        ports:
          - "80:3000"
        env:
          ENVIRONMENT: production
          LOG_LEVEL: info
        restart_policy: always
    
    - name: Install monitoring agent
      apt:
        name: datadog-agent
        state: present
    
    - name: Configure Datadog agent
      template:
        src: datadog-agent.conf.j2
        dest: /etc/datadog-agent/datadog.yaml
        owner: root
        group: root
        mode: '0644'
      notify: Restart Datadog
    
    - name: Enable and start Datadog
      systemd:
        name: datadog-agent
        enabled: yes
        state: started
    
    - name: Install log collection agent
      apt:
        name: filebeat
        state: present
    
    - name: Configure Filebeat
      template:
        src: filebeat.yml.j2
        dest: /etc/filebeat/filebeat.yml
      notify: Restart Filebeat
    
    - name: Verify Docker is running
      shell: docker ps
      register: docker_status
      failed_when: docker_status.rc != 0
    
    - name: Health check - verify app is responding
      uri:
        url: http://localhost:80/health
        method: GET
        status_code: 200
      retries: 5
      delay: 10
  
  handlers:
    - name: Restart Datadog
      systemd:
        name: datadog-agent
        state: restarted
    
    - name: Restart Filebeat
      systemd:
        name: filebeat
        state: restarted
```

### Run the Playbook
```bash
# Provision all 100 servers
ansible-playbook -i inventory/aws_ec2.yml playbooks/provision.yml

# Provision only web servers
ansible-playbook -i inventory/aws_ec2.yml -l web_servers playbooks/provision.yml

# Dry run (preview changes)
ansible-playbook -i inventory/aws_ec2.yml playbooks/provision.yml --check
```

### Result
- 100 servers provisioned in 5-10 minutes
- Identical configuration on all servers
- Fully automated and repeatable
- All steps logged for audit trail

---

## Real-Time Scenario 2: Zero-Downtime Application Update

### Scenario Description
Deploy new application version to 20 servers in a load-balanced environment with rolling updates.

### Problem
- Stop all servers → Deploy → Start (causes downtime)
- Need to update one server at a time
- Remove from load balancer → Deploy → Verify → Add to load balancer

### Ansible Solution

```yaml
# playbooks/deploy.yml
---
- name: Rolling application deployment
  hosts: web_servers
  serial: 1  # Update one server at a time
  
  tasks:
    - name: Get current load balancer status
      uri:
        url: "http://lb.internal/api/servers/{{ inventory_hostname }}"
        method: GET
      register: lb_status
    
    - name: Remove server from load balancer
      uri:
        url: "http://lb.internal/api/servers/{{ inventory_hostname }}/drain"
        method: POST
      when: lb_status.json.active
    
    - name: Wait for active connections to drain
      pause:
        seconds: 30
    
    - name: Stop application
      docker_container:
        name: myapp
        state: stopped
    
    - name: Pull new application image
      docker_image:
        name: "myapp:{{ new_version }}"
        source: registry
        force_source: yes
    
    - name: Start application with new version
      docker_container:
        name: myapp
        image: "myapp:{{ new_version }}"
        ports:
          - "80:3000"
        restart_policy: always
    
    - name: Wait for application to be ready
      uri:
        url: "http://localhost:80/health"
        method: GET
        status_code: 200
      retries: 10
      delay: 5
    
    - name: Run smoke tests
      shell: |
        curl -f http://localhost:80/api/status || exit 1
        curl -f http://localhost:80/api/version || exit 1
      register: smoke_test
      failed_when: smoke_test.rc != 0
    
    - name: Add server back to load balancer
      uri:
        url: "http://lb.internal/api/servers/{{ inventory_hostname }}/activate"
        method: POST
    
    - name: Verify server is healthy in load balancer
      uri:
        url: "http://lb.internal/api/servers/{{ inventory_hostname }}"
        method: GET
      until: result.json.status == 'healthy'
      retries: 5
      delay: 5
    
    - name: Wait before updating next server
      pause:
        seconds: 10
```

### Deploy
```bash
ansible-playbook -i inventory/aws_ec2.yml playbooks/deploy.yml -e "new_version=2.0.0"
```

### Timeline
```
Server 1: Drain → Stop → Deploy → Start → Verify → Activate (3 min)
Server 2: Drain → Stop → Deploy → Start → Verify → Activate (3 min)
...
Server 20: Drain → Stop → Deploy → Start → Verify → Activate (3 min)

Total: ~60 minutes for 20 servers
Downtime: 0 minutes (requests redirect to other servers)
```

---

## Real-Time Scenario 3: Incident Response Automation

### Scenario Description
When servers have high disk usage, automatically clean up old logs without manual intervention.

### Ansible Solution

```yaml
# playbooks/incident-response.yml
---
- name: Automated incident response - high disk usage
  hosts: web_servers
  
  tasks:
    - name: Check disk usage
      shell: df -h / | grep '/' | awk '{print $5}' | sed 's/%//g'
      register: disk_usage
    
    - name: Alert if disk > 85%
      debug:
        msg: "WARNING: Disk usage at {{ disk_usage.stdout }}%"
      when: disk_usage.stdout | int > 85
    
    - name: Clean old logs if disk > 90%
      shell: |
        find /var/log -type f -name "*.log" -mtime +30 -delete
        find /var/log -type f -name "*.gz" -mtime +30 -delete
      when: disk_usage.stdout | int > 90
    
    - name: Clear container logs if disk > 90%
      shell: |
        truncate -s 0 /var/lib/docker/containers/*/*.log
      when: disk_usage.stdout | int > 90
    
    - name: Create incident ticket
      shell: |
        curl -X POST http://ticketing.internal/api/tickets \
          -H 'Content-Type: application/json' \
          -d '{
            "title": "High disk usage on {{ inventory_hostname }}",
            "severity": "medium",
            "host": "{{ inventory_hostname }}",
            "disk_usage": "{{ disk_usage.stdout }}%"
          }'
      when: disk_usage.stdout | int > 85
    
    - name: Notify Slack
      slack:
        token: "{{ slack_token }}"
        msg: "Disk cleanup performed on {{ inventory_hostname }}: {{ disk_usage.stdout }}% → {{ new_disk_usage.stdout }}%"
        channel: "#ops"
      when: disk_usage.stdout | int > 90
```

### Run as Cron Job
```bash
# Run every hour
0 * * * * ansible-playbook -i /etc/ansible/inventory /etc/ansible/playbooks/incident-response.yml
```

---

## Real-Time Scenario 4: Configuration Compliance Checking

### Scenario Description
Ensure all servers comply with security policies:
- SSH key-based auth only (no passwords)
- Firewall enabled
- Fail2ban installed
- NTP configured

### Ansible Solution

```yaml
# playbooks/compliance-check.yml
---
- name: Security compliance check
  hosts: web_servers
  
  tasks:
    - name: Check SSH password auth is disabled
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present
      notify: Restart SSH
    
    - name: Ensure UFW is enabled
      ufw:
        state: enabled
    
    - name: Install Fail2ban
      apt:
        name: fail2ban
        state: present
    
    - name: Start Fail2ban
      systemd:
        name: fail2ban
        enabled: yes
        state: started
    
    - name: Install NTP
      apt:
        name: chrony
        state: present
    
    - name: Start NTP
      systemd:
        name: chrony
        enabled: yes
        state: started
    
    - name: Generate compliance report
      template:
        src: compliance-report.j2
        dest: /var/log/compliance-report-{{ ansible_date_time.iso8601 }}.txt
    
    - name: Send compliance report
      mail:
        host: mail.internal
        to: security@company.com
        subject: "Compliance Report - {{ inventory_hostname }}"
        body: "{{ lookup('file', '/var/log/compliance-report.txt') }}"
  
  handlers:
    - name: Restart SSH
      systemd:
        name: ssh
        state: restarted
```

---

## Ansible Concepts

### Playbook
- YAML file with list of tasks
- Executed sequentially on hosts
- Can include variables, conditionals, loops

### Inventory
- List of target hosts
- Can be static (file) or dynamic (script)
- Supports grouping and variables

### Module
- Unit of work (apt, docker, uri, shell, etc.)
- Idempotent (safe to run multiple times)
- Returns state information

### Handler
- Triggered by "notify"
- Runs once at end of play
- Used for service restarts

### Idempotency
- Running playbook 10x = same result
- Modules check if change is needed
- Safe for automation

---

## When to Use Ansible

✅ **Good Use Cases:**
- Server configuration management
- Application deployment
- Orchestration tasks
- Ad-hoc administration
- Multi-step workflows
- Agentless automation

❌ **Avoid If:**
- Simple one-off tasks (just SSH)
- Very frequent updates (agents better)
- Complex state machines

---

## Learning Path
1. Learn YAML syntax
2. Create first playbook
3. Practice with inventory
4. Master conditionals and loops
5. Learn handlers and notifications
6. Build reusable roles
7. Understand idempotency
