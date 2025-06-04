[Русская версия](./README.ru.md)

This guide provides instructions for running Ansible playbooks for VPS configure and Vmess install.

### Notes

- The VMess server is deployed using the official Docker image: [`v2fly/v2fly-core`](https://hub.docker.com/r/v2fly/v2fly-core).

## Infrastructure Requirements

Before you start, make sure you have the following:

- **Ansible host:** The machine where you will run Ansible playbooks. It must have Ansible and Python 3.x installed.
- **VPS host:** The target server that will be provisioned and configured (VMess, Docker, Nginx, etc.).
- **Domain or subdomain:** Required for SSL certificate issuance via Certbot and for Nginx/VMess setup.
- **Prometheus host (optional):** Server for collecting metrics from your VPS hosts via Node Exporter.
- **Inventory file:** Ansible inventory file describing your infrastructure and hosts. You must have access to this file and any required credentials.


## Directory Structure

```
.
├── add_user.yml
├── delete_user.yml
├── generate_config.yml
├── README.md
├── vmess_init_and_configure.yml
└── roles
    ├── add_user
    │   ├── defaults
    │   │   ├── main_template.yml
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── create_bootstrap_user
    │   ├── defaults
    │   │   ├── main_template.yml
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── delete_user
    │   ├── defaults
    │   │   ├── main_template.yml
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── generate_config
    │   ├── defaults
    │   │   ├── main_template.yml
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── config_template.j2
    └── vmess_init_and_configure
        ├── defaults
        │   ├── main_template.yml
        │   └── main.yml
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   ├── install_base_vmess.yml
        │   ├── install_docker.yml
        │   ├── install_fail2ban.yml
        │   ├── install_nodeexporter.yml
        │   ├── install_ufw.yml
        │   └── main.yml
        └── templates
            ├── base_config.json.j2
            ├── docker-compose.yml.j2
            ├── jail.local.j2
            ├── nginx_base.conf.j2
            ├── nginx.conf.j2
            ├── nginx_temp.conf.j2
            ├── node_exporter.service.j2
            └── prometheus_job.j2
```

## Main Roles

- **create_bootstrap_user:** Creates a bootstrap user
- **vmess_init_and_configure:** install and configures:
    - UFW, 
    - Node Exporter, 
    - Fail2Ban,
    - Docker
    - Nginx with certbot,
    - Vmess
- **add_user:** Role for creating vmess client
- **delete_user:** Role for deleting vmess client
- **generate_config:** Role for creating vmess client config

## How to Run a Playbook

1. **Navigate to the project directory:**
   ```bash
   cd ./VMESS
   ```

2. **Run a playbook:**
   ```bash
   ansible-playbook -i inventory/your_inventory your_playbook.yml
   ```
   - Replace `your_inventory` with your inventory file.
   - Replace `your_playbook.yml` with the playbook you want to run.

3. **Optional: Specify extra variables**
   ```bash
   ansible-playbook -i inventory/your_inventory your_playbook.yml -e '{"key": "value"}'
   ```

## How to configure target server
> **Important:** Only run `vmess_init_and_configure.yml` for a single host from your inventory at a time (with `-l target_host`). Do not run it for multiple hosts simultaneously.

1. **Run a playbook:**
   ```bash
   ansible-playbook -i inventory/your_inventory vmess_init_and_configure.yml -l targer_host --ask-pas
   ```
   - Replace `your_inventory` with your inventory file.
   - Replace `your_playbook.yml` with the playbook you want to run.
   - Replace `targer_host` with target host

2. **Optional: Specify extra variables**
   ```bash
   ansible-playbook -i inventory/your_inventory vmess_init_and_configure.yml -e '{"key": "value"}' --ask-pas
   ```
    You can provide variables for the playbook in two ways:
    - Define them in `roles/vmess_init_and_configure/defaults/main.yml` (they will be used by default).
    - Override or set them at runtime using the `-e` option when running the playbook. You can pass variables as a string (`-e "key=value"`) or as a JSON object (`-e '{"key": "value"}'`). Variables passed via `-e` take precedence over values in `defaults/main.yml`.

    Below are the main variables used for configuring your VMess server and related services. These variables are defined by default in `roles/vmess_init_and_configure/defaults/main.yml`, but you can override them at runtime using the `-e` option.

### VM with VMess

- **ssh_port**: SSH port on the VM. Should match `ansible_port` in your inventory.
- **server_name**: Domain name or subdomain for VMess and SSL certificate (used by Nginx and Certbot).
- **certbot_email**: Email address for Certbot to register and renew SSL certificates.
- **nginx_conf_path**: Path where the Nginx configuration for VMess will be stored on the VM.
- **docker_dir**: Directory on the VM where the Docker Compose file and VMess config will be stored.
- **fail2ban_ignoreip**: List of IP addresses to ignore in Fail2Ban for SSH protection (space-separated).
- **node_exporter_allowed_src_ip**: IP address allowed to access Node Exporter metrics (usually your Prometheus server).

### VM/Prometheus Host

- **prometheus_conf_paths**: Path to the Prometheus configuration file on the Prometheus host.
- **prometheus_host**: Name of the Prometheus host as defined in your Ansible inventory.

### Variables for Optional Installs

- **install_fail2ban**: install Fail2Ban (`true` or `false`).
- **install_nodeexporter**: install Node Exporter (`true` or `false`).
- **install_ufw**: install UFW firewall (`true` or `false`).
- **install_docker**: install Docker and Docker Compose (`true` or `false`).
- **setup_prometheus**: Add scrape-config in prometheus config file on extrernal server (`true` or `false`).
---

## How to add vmess client to vmess server config
> **Note:** You can run `add_user.yml` for multiple VMess hosts simultaneously by specifying a group or multiple hosts in your inventory.  
> **Important:** All target hosts must have the same VMess configuration paths (`vmess_config_file_path`, `docker_compose_dir`) to ensure correct operation of the playbook.

1. **Run a playbook:**
   ```bash
   ansible-playbook -i inventory/your_inventory add_user.yml -l targer_host
   ```
   - Replace `your_inventory` with your inventory file.
   - Replace `your_playbook.yml` with the playbook you want to run.
   - Replace `targer_host` with target host (hosts group)

2. **Optional: Specify extra variables**
   ```bash
   ansible-playbook -i inventory/your_inventory add_user.yml -e '{"key": "value"}'
   ```
    You can provide variables for the playbook in two ways:
    - Define them in `roles/add_user/defaults/main.yml` (they will be used by default).
    - Override or set them at runtime using the `-e` option when running the playbook. You can pass variables as a string (`-e "key=value"`) or as a JSON object (`-e '{"key": "value"}'`). Variables passed via `-e` take precedence over values in `defaults/main.yml`.

    Below are the main variables used for configuring your VMess server and related services. These variables are defined by default in `roles/add_user/defaults/main.yml`, but you can override them at runtime using the `-e` option.

### Variables for add_user role

- **vmess_config_file_path**: Path to the VMess server configuration file. The user will be added to this config.
- **docker_compose_dir**: Path to the directory containing the Docker Compose file and related VMess configuration files.
- **email**: Name or identifier for the new VMess user (used as a label or for client identification).
---

## How to delete vmess client to vmess server config
> **Note:** You can run `delete_user.yml` for multiple VMess hosts simultaneously by specifying a group or multiple hosts in your inventory.  
> **Important:** All target hosts must have the same VMess configuration paths (`vmess_config_file_path`, `docker_compose_dir`) to ensure correct operation of the playbook.

1. **Run a playbook:**
   ```bash
   ansible-playbook -i inventory/your_inventory delete_user.yml -l targer_host
   ```
   - Replace `your_inventory` with your inventory file.
   - Replace `your_playbook.yml` with the playbook you want to run. 
   - Replace `targer_host` with target host (hosts group)

2. **Optional: Specify extra variables**
   ```bash
   ansible-playbook -i inventory/your_inventory delete_user.yml -e '{"key": "value"}'
   ```
    You can provide variables for the playbook in two ways:
    - Define them in `roles/delete_user/defaults/main.yml` (they will be used by default).
    - Override or set them at runtime using the `-e` option when running the playbook. You can pass variables as a string (`-e "key=value"`) or as a JSON object (`-e '{"key": "value"}'`). Variables passed via `-e` take precedence over values in `defaults/main.yml`.

    Below are the main variables used for configuring your VMess server and related services. These variables are defined by default in `roles/delete_user/defaults/main.yml`, but you can override them at runtime using the `-e` option.

### Variables for delete_user role

- **vmess_config_file_path**: Path to the VMess server configuration file. The user will be added to this config.
- **docker_compose_dir**: Path to the directory containing the Docker Compose file and related VMess configuration files.
- **email**: Name or identifier for the new VMess user (used as a label or for client identification).
---

## How to generate vmess client config
> **Note:** You can run `generate_config.yml` for multiple VMess hosts simultaneously by specifying a group or multiple hosts in your inventory.  
> **Important:** All target hosts must have the same VMess configuration paths (`vmess_config_file_path`, `docker_compose_dir`) to ensure correct operation of the playbook.

1. **Run a playbook:**
   ```bash
   ansible-playbook -i inventory/your_inventory generate_config.yml -l targer_host
   ```
   - Replace `your_inventory` with your inventory file.
   - Replace `your_playbook.yml` with the playbook you want to run.
   - Replace `targer_host` with target host (hosts group)

2. **Optional: Specify extra variables**
   ```bash
   ansible-playbook -i inventory/your_inventory deletgenerate_confige_user.yml -e '{"key": "value"}'
   ```
    You can provide variables for the playbook in two ways:
    - Define them in `roles/generate_config/defaults/main.yml` (they will be used by default).
    - Override or set them at runtime using the `-e` option when running the playbook. You can pass variables as a string (`-e "key=value"`) or as a JSON object (`-e '{"key": "value"}'`). Variables passed via `-e` take precedence over values in `defaults/main.yml`.

    Below are the main variables used for configuring your VMess server and related services. These variables are defined by default in `roles/generate_config/defaults/main.yml`, but you can override them at runtime using the `-e` option.

### Variables for generate_config role

- **vmess_config_file_path**: Path to the VMess server configuration file from which the client config will be generated.
- **email**: Name or identifier for the VMess user (used as a label or for client identification).

#### Password Push Integration

If you want to automatically publish the generated config and receive a link using [Password Pusher](https://pwpush.com/):

- **pwd_push**: Set to `true` to enable publishing the config via Password Pusher.
- **pwpush_url**: URL of your Password Pusher instance.
- **pwpush_email**: Email for authentication with Password Pusher (user/admin).
- **pwpush_api_token**: API token for authentication with Password Pusher.

If `pwd_push` is enabled, the playbook will send the generated config to the specified Password Pusher service and return a link for sharing. If `pwd_push` is disabled or set to `false`, the generated client configuration will be displayed directly in the console output.

---

## License

This project is licensed under the [MIT License](./LICENSE).

You are free to use, modify, and distribute this software for any purpose, including commercial use, provided that the original copyright notice and license text are included in all copies.

See the [LICENSE](./LICENSE) file for full details.