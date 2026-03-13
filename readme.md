# Playbooks 📒
![](https://img.shields.io/gitlab/pipeline-status/engineering-with-ai/tooling-playbooks?branch=main&logo=gitlab)
![](https://img.shields.io/badge/ansible-gray?logo=ansible)

Ansible playbooks to set up infrastructure prerequisites, including development services and a GitLab runner.

## Services Included
- PostgreSQL (daemon)
- Redis (daemon)
- Neo4j (daemon)
- MinIO (daemon)
- K3s (daemon)
- Ollama (daemon)
    - llama - conversational
    - qwen-coder - code generation
    - nomic - embeddings
- Mender (docker)
- MLflow (docker)
- F-Droid server (docker)
- Harbor (docker)
  - add  to `/etc/docker/daemon.json`:
    ```json
      {
        "insecure-registries": ["your-ip:8083"]
      }
    ```
- Prometheus (docker)
- Grafana (docker)

## Deployment Diagram

```plantuml
rectangle users #line.dashed {
        rectangle admin {
                rectangle monitoring #line.dashed {
                    rectangle prometheus
                    rectangle grafana
                }

                rectangle database_daemons #line.dashed {
                    database postgres
                    database neo4j
                    database redis
                    database minio
                }
                rectangle docker_artifact_repos #line.dashed {
                    rectangle harbor
                    rectangle fdroid
                    rectangle mender
                    rectangle mlflow
                }
                rectangle infrastructure #line.dashed {
                    rectangle k3s
                    rectangle docker
                }
                rectangle ai_stack #line.dashed {
                    rectangle ollama_daemon
                    rectangle qwen_coder
                    rectangle nomic_embed
                }
        }
        rectangle gitlab_runner {
            rectangle ci_python_tooling
            rectangle ci_bun_tooling
            rectangle ci_rust_tooling
        }
}
```
## Requirements

- Target machine running a Debian-based system (e.g., Ubuntu)
- ansible-playbook installed on the control machine
- SSH access to the target machine
- Sudo access on the target machine

## Setup

1. Copy `sample.secrets.yml` to `secrets.yml` and fill in your values:
   ```
   cp sample.secrets.yml secrets.yml
   ```

2. Copy `sample.inventory.ini` to `inventory.ini` and update with your target machine details:
   ```
   cp sample.inventory.ini inventory.ini
   ```

## Usage

Run the main playbook:

```
ansible-galaxy collection install community.postgresql
ansible-playbook -i inventory.ini main.yaml
```

You can choose to set up development services, GitLab runner, or both during the execution.

## EC2 Configuration

For those using Amazon EC2, here's the minimum configuration to run all services:

```bash
aws ec2 run-instances \
    --image-id ami-064519b8c76274859 \
    --count 1 \
    --instance-type c5.xlarge \
    --key-name foo \
    --security-group-ids sg-0000 \
    --subnet-id subnet-0000 \
    --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":
{"VolumeSize":24,"VolumeType":"gp2"}}]'
```

This configuration uses:
- An Amazon Linux 2 AMI (adjust `--image-id` for your region)
- c5.xlarge instance type (4 vCPU, 8 GiB memory)
- 24 GB GP2 EBS volume

Note: Adjust the `--key-name`, `--security-group-ids`, and `--subnet-id` according to your AWS setup.

Ensure that your security group allows inbound SSH access (port 22) from your IP address.

**Important:** As of 2025, a c5.xlarge instance costs approximately `$0.17 per hour`. This can add up to about $125 per month if run continuously. For long-term use, a self-hosted server is strongly recommended as it can be significantly more cost-effective, especially for continuous operation.



## Notes

- These playbooks make significant changes to the system. Use with caution and preferably on a fresh installation.
- Ensure you have backups and understand the changes being made before running the playbooks.
- The playbooks are designed for Debian-based systems and may need modifications for other distributions.
- Some tasks may take a while to complete, mostly python installation and setting up deepseek

