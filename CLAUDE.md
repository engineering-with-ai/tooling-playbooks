# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains Ansible playbooks for setting up infrastructure prerequisites for the [fullstack.energy](http://fullstack.energy) project. The playbooks configure development services and GitLab runners on Debian-based systems, particularly optimized for Ubuntu and Amazon EC2 instances.

## Architecture

The playbooks follow a modular structure:

- `main.yml`: Entry point that handles common dependencies (Docker) and orchestrates other playbooks
- `dev-services-setup.yml`: Configures development infrastructure including databases, storage, monitoring, and ML services
- `gitlab-runner-setup.yml`: Sets up GitLab CI/CD runners with development toolchains
- `templates/`: Jinja2 templates for service configurations and Docker Compose files
- `secrets.yml` / `inventory.ini`: Environment-specific configuration files 

### Service Stack Architecture

**Development Services (dev-services-setup.yml):**
- **Databases**: PostgreSQL 15 with TimescaleDB and PGVector extensions, Neo4j
- **Storage**: MinIO object storage with pre-configured buckets
- **Container Orchestration**: K3s (lightweight Kubernetes)
- **AI/ML**: Ollama with qwen3-coder:30b, llama3.1:70b (conversational), and nomic-embed-text models
- **Artifact Repositories**: Mender, F-Droid server, Harbor
- **Monitoring**: Prometheus + Grafana stack
- **ML Tracking**: MLflow with PostgreSQL backend

**GitLab Runner Environment:**
- Multi-language toolchain: Rust (1.85.0), Python (3.13.2), Node.js (22.x)
- Package managers: Cargo, pip with version managers (pyenv)
- Specialized Rust tools: cargo-audit, cargo-udeps, cargo-espflash, etc.


### Quick Fixes Needed:
- Add `changed_when: false` to read-only operations
- Use `creates:` parameter for installation tasks
- Add proper state checking before expensive operations (model pulls, tool installs)
- Replace shell scripts with dedicated Ansible modules where possible
- Add conditional checks for service configuration changes

## Common Commands

### Initial Setup
```bash
# Copy configuration templates
cp sample.secrets.yml secrets.yml
cp sample.inventory.ini inventory.ini

# Install required Ansible collections
ansible-galaxy collection install community.postgresql

# Run full setup (will prompt for setup type)
ansible-playbook -i inventory.ini main.yml
```

### Running Specific Components
```bash
# Development services only
ansible-playbook -i inventory.ini main.yml --extra-vars "setup_type=dev-services"

# GitLab runner only
ansible-playbook -i inventory.ini main.yml --extra-vars "setup_type=gitlab-runner"

# Both (default)
ansible-playbook -i inventory.ini main.yml --extra-vars "setup_type=both"
```

### Debugging Slow Runs
```bash
# Run with increased verbosity to see what's changing
ansible-playbook -i inventory.ini main.yml -v

# Skip expensive operations during development
ansible-playbook -i inventory.ini main.yml --skip-tags ollama,models,cargo_tools

# Check what would change without applying
ansible-playbook -i inventory.ini main.yml --check --diff
```

### Service Management
```bash
# Check service status on target machine
systemctl status postgresql minio neo4j ollama gitlab-runner

# View service logs
journalctl -u <service_name> -f

# Restart services
sudo systemctl restart <service_name>
```

### Docker Compose Services
All containerized services are located in `/home/admin/docker/` on the target machine:
```bash
# View all running containers
docker ps

# Restart a service stack
cd /home/admin/docker/<service_name>
docker compose restart

# View service logs
docker compose logs -f <service_name>
```

## Ansible Best Practices for This Repo

### Immediate Improvements Needed:

1. **Add Idempotency Guards**:
```yaml
# BAD: Runs every time
- name: Pull Ollama model
  shell: ollama pull {{ embedding_model }}

# GOOD: Check if model exists first
- name: Check if Ollama model exists
  shell: ollama list | grep {{ embedding_model }}
  register: model_check
  failed_when: false
  changed_when: false

- name: Pull Ollama model
  shell: ollama pull {{ embedding_model }}
  when: model_check.rc != 0
```

2. **Use Proper Ansible Modules**:
```yaml
# BAD: Shell script for Docker Compose
- name: Start service
  command: docker compose up -d

# GOOD: Use community.docker.docker_compose
- name: Start service
  community.docker.docker_compose:
    project_src: /path/to/compose
    state: present
```

3. **Add Tags for Selective Running**:
```yaml
- name: Expensive operation
  shell: long_running_command
  tags: ['expensive', 'models']
```

4. **Proper Changed Detection**:
```yaml
- name: Configure service
  lineinfile:
    path: /etc/service.conf
    line: "setting=value"
  register: config_changed

- name: Restart service
  systemd:
    name: service
    state: restarted
  when: config_changed.changed
```

## Configuration Management

### Version Management
Key versions are defined in `main.yml:vars` and should be updated there:
- `rust_version`: Rust toolchain version
- `python_version`: Python version via pyenv
- `postgres_version`: PostgreSQL major version
- `coding_llm_name`: Coding-focused LLM (qwen3-coder:30b)
- `conversational_llm_name`: Domain expert conversational LLM (llama3.1:70b)
- `embedding_model`: Text embedding model (nomic-embed-text:v1.5)
- Service versions for Mender, Prometheus, Grafana, Harbor, MLflow

### Service Ports
Standard port mappings (configurable in templates):
- MinIO: 9000, 9001 (console)
- PostgreSQL: 5432
- Neo4j: 7474 (HTTP), 7687 (Bolt)
- MLflow: 5000
- F-Droid: 8666
- Harbor: 8083
- Prometheus: 9090
- Grafana: 3000
- Ollama: 11434

## EC2 Deployment Notes

Minimum recommended configuration:
- Instance type: c5.xlarge (4 vCPU, 8 GiB RAM)
- Storage: 24 GB GP2 EBS volume
- Security groups must allow SSH (port 22) and service ports as needed
- Estimated cost: ~$0.17/hour (~$125/month if run continuously)

## Development Workflow

1. **Before Making Changes**: Test with `--check --diff` to see what would change
2. **Add Proper Idempotency**: Ensure tasks only run when needed
3. **Use Tags**: Tag expensive operations for selective execution
4. **Test Incrementally**: Run playbook multiple times to ensure idempotency
5. **Update Version Variables**: Modify `main.yml` when upgrading dependencies

## Important Notes
- **Recommended**: Run with specific tags to avoid expensive re-runs during development
- Services configure remote access - ensure proper network security
- Setup optimized for development/testing, not production hardening