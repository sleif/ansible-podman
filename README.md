# sleif.podman

This role installs and manages Podman containers, pods, networks, and related systemd integration. It supports both rootfull (root) and rootless (non-root) operations.

The role implements the following operations:

```yaml
podman_operations:
  - podman_container_create
  - podman_image_build
  - podman_image_pull
  - podman_init_vars
  - podman_install
  - podman_network_create
  - podman_socket_crud
  - podman_pod_create
  - podman_system_prune
  - podman_systemd_restart_pod_or_container
```

## Supported Operations

- **podman_install**: Installs Podman and required dependencies, sets up the podman user and group for rootless operations.
- **podman_init_vars**: Initializes environment variables for other roles. Sets and returns:
  - `_container_storage_dir_base_local` - local container storage directory
  - `_container_storage_dir_base` - base container storage directory
  - `_group` - group for rootless/rootful operations
  - `_owner` - owner for rootless/rootful operations
  - `_systemd_scope` - `user` (rootless) or `system` (rootful)
  - `_systemd_service_files_dir` - directory for systemd unit files
  - `_quadlet_files_dir` - directory for Podman quadlet files
  - `_xdg_runtime_dir` - XDG runtime directory
  - `_container_run_as_uid` - UID of the configured user
- **podman_pod_create**: Creates a Podman pod with optional networks. Must include `pod_name` variable.
- **podman_container_create**: Creates a Podman container within a pod or standalone. Must include `container_name` or `_container` variable. Automatically creates networks if needed and supports socket creation.
- **podman_network_create**: Creates Podman networks defined in `podman_networks` dictionary. Works in proper rootful/rootless context. Networks defined in `podman_install` operation are not auto-created.
- **podman_socket_crud**: Creates systemd socket units for containers to enable rootless access to host ports without network bridging (e.g., Caddy server on port 80/443).
- **podman_image_build**: Builds container images using podman. Supports custom temporary directories to avoid `/var/tmp` constraints.
- **podman_image_pull**: Pulls container images from registries.
- **podman_system_prune**: Cleans up unused images, containers, volumes, and networks. Can be configured with minimum free disk space threshold.
- **podman_systemd_restart_pod_or_container**: Manages systemd unit files for pods and containers. Required for proper systemd integration. Containers must be created with `state: created` before calling this operation.

## Requirements

- Ansible >= 2.15
- `containers.podman` collection (for `containers.podman.*` modules)
- EL 9 (Enterprise Linux 9) or compatible distribution
- Root or sudo access for `podman_install` operation
- For rootless operations: a non-root user account (created if not present)

## Role Variables

All variables are defined in `defaults/main.yml` with sensible defaults for rootless operation.

### Required Variables

- **podman_operation**: (string) The operation to execute. Must be one of the supported operations listed above. *No default.*

### Core Configuration

- **podman_rootless**: (boolean, default: `true`) - Whether to run Podman in rootless mode (non-root user) or rootfull mode (root).
- **podman_user**: (string, default: `podman`) - The user account for rootless Podman operations. Created automatically if it doesn't exist.
- **podman_group**: (string, default: `podman`) - The group for rootless Podman operations. Created automatically if it doesn't exist.
- **podman_user_home**: (string, default: `/srv/podman`) - Home directory for the podman user.
- **container_storage_dir_base_local**: (string, default: `/srv`) - Base directory for container storage on the local system.
- **container_storage_dir_base**: (string, default: `/srv`) - Base directory for container storage (can be remote).

### Network Configuration

- **podman_networks**: (dict, default: `{}`) - Dictionary of custom networks to create. Each network requires:
  - `name`: (string) Network name
  - `subnet`: (string) Subnet in CIDR format (e.g., `10.0.0.0/24`)
  - `gateway`: (string) Gateway IP address
  - `iprange`: (string, optional) IP range within the subnet
  
  Example:
  ```yaml
  podman_networks:
    custom_net:
      name: 'custom_network'
      subnet: '10.99.0.0/16'
      gateway: '10.99.0.1'
      iprange: '10.99.0.128/25'
  ```

- **podman_network_names**: (list, default: `[]`) - List of additional network names to attach to containers. Can include `slirp4netns` or `pasta` for special networking modes.

### Logging and Storage

- **podman_log_driver**: (string, default: `k8s-file`) - Log driver for rootless Podman containers. Alternatives: `json-file`, `journald`, etc.
- **podman_image_copy_tmp_dir_root**: (string, optional) - Custom temporary directory for rootfull image building to avoid `/var/tmp` space constraints.
- **podman_image_copy_tmp_dir_rootless**: (string, optional) - Custom temporary directory for rootless image building to avoid `/var/tmp` space constraints.

### System Configuration

- **net_core_rmem_max**: (string, default: `7500000`) - Maximum receive socket buffer size for network operations.
- **net_core_wmem_max**: (string, default: `7500000`) - Maximum send socket buffer size for network operations.

### System Prune Configuration

- **podman_system_prune**: (boolean, default: `false`) - Whether to enable automatic system pruning.
- **podman_system_prune_min_free_gb**: (integer, default: `5`) - Minimum free disk space in GB. If available space falls below this, prune is executed.
- **podman_system_prune_force**: (boolean, default: `false`) - Whether to force prune even non-dangling images.

### Package Configuration

- **podman_packages**: (list) - List of packages to install. Default includes:
  - `aardvark-dns` - DNS plugin for pod networking
  - `bash-completion` - Bash completions for Podman
  - `conmon` - Container monitor
  - `container-selinux` - SELinux policies for containers
  - `crun` - OCI runtime
  - `fuse-overlayfs` - Overlay filesystem driver for rootless
  - `netavark` - Network plugin
  - `podman` - Container runtime
  - `podman-plugins` - Podman plugins
  - `shadow-utils` - User namespace tools
  - `slirp4netns` - User-mode networking
  - `passt` - Pasta for networking

## Operation-Specific Variables

### Container Creation (`podman_container_create`)

- **container_name**: (string) - Name of the container to create.
- **pod_name**: (string, optional) - If specified, container will be created within this pod.
- **_container**: (dict) - Container configuration (passed internally).
- **volumes**: (list, optional) - List of volume mounts with `host` and `container` keys.
- **secrets**: (list, optional) - List of secrets with `name` and `data` keys.
- **hostname**: (string, optional) - Hostname for the container.

### Network Creation (`podman_network_create`)

- **network**: (dict) - Network configuration with `name`, `subnet`, `gateway`, and optional `iprange`.

### Pod Creation (`podman_pod_create`)

- **pod_name**: (string) - Name of the pod to create.

### Socket Creation (`podman_socket_crud`)

- **sockets**: (dict) - Socket configuration with:
  - `container`: (string) - Target container name
  - `sockets`: (list) - List of socket listener specifications (e.g., `ListenStream=[::]:80`, `ListenDatagram=[::]:443`)

### Image Operations (`podman_image_pull`, `podman_image_build`)

- **image_name**: (string) - Name of the image to pull or build.
- **image_tag**: (string, optional, default: `latest`) - Tag for the image.
- **image_tags_to_remove**: (list, optional) - List of specific image tags to remove. Useful for cleaning up old or outdated images by tag (e.g., `['myapp:old-tag', 'nginx:1.0']`). Used with `podman_image_pull` operation.

## Dependencies

None - this role is self-contained. However, it is commonly used to support other roles that require Podman infrastructure. See the "Initialize Podman Environment Variables" example for how to use this role from other roles.

## Example Playbook

### 1. Install Podman

```yaml
- name: Install Podman
  hosts: "all"
  user: root

  roles:
    - {role: sleif.podman, tags: "podman_install",
       podman_operation: "podman_install"}
```

### 2. Initialize Podman Environment Variables

Use this when calling the role from other roles to get environment paths and configuration:

```yaml
- name: Include podman_init_vars from sleif.podman
  ansible.builtin.include_tasks: "{{ lookup('first_found', params) }}"
  vars:
    params:
      files: sleif.podman/tasks/includes/podman_init_vars.yml
      paths: "{{ lookup('config', 'DEFAULT_ROLES_PATH') }}"
  tags: always
```

See the [Supported Operations](#supported-operations) section under **podman_init_vars** for the complete list of facts that will be set.

### 3. Create a Podman Pod with Networks

```yaml
- name: Create a Podman pod with custom networks
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_pod_create
    podman_rootless: true
    pod_name: 'my_app_pod'
    podman_networks:
      app_network:
        name: 'app_network'
        subnet: '10.0.0.0/24'
        gateway: '10.0.0.1'
        iprange: '10.0.0.128/25'
      database_network:
        name: 'database_network'
        subnet: '10.1.0.0/24'
        gateway: '10.1.0.1'
        iprange: '10.1.0.128/25'
  tags: podman_pod_create
```

### 4. Create a Container

```yaml
- name: Create a container in a pod
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_container_create
    target: "{{ pod_name if pod_name | d('') is truthy else container_name }}"
    container_name: 'web_server'
    hostname: 'web-server-1'
    podman_networks:
      app_network:
        name: 'app_network'
        subnet: '10.0.0.0/24'
        gateway: '10.0.0.1'
        iprange: '10.0.0.128/25'
    volumes:
      - {'host': '/srv/podman/web_server/data', 'container': '/app/data'}
      - {'host': '/srv/podman/web_server/config', 'container': '/app/config'}
    secrets:
      - {'name': 'api_key', 'data': '{{ secret_api_key }}'}
      - {'name': 'db_password', 'data': '{{ secret_db_password }}'}
  tags: podman_container_create
```

### 5. Create Container with Socket Units (Rootless Port Access)

For rootless containers to listen on host ports without networking (e.g., Caddy):

```yaml
- name: Create Caddy container with socket units
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_container_create
    container_name: 'caddy'
    hostname: 'caddy'
    podman_networks:
      app_network:
        name: 'app_network'
        subnet: '10.0.0.0/24'
        gateway: '10.0.0.1'
    podman_sockets:
      container: 'caddy'
      sockets:
        - 'ListenStream=[::]:80'     # fd/3 in container
        - 'ListenStream=[::]:443'    # fd/4 in container
        - 'ListenDatagram=[::]:443'  # fdgram/5 in container
        - 'ListenStream=[::1]:2019'  # fd/6 in container (admin API)
    volumes:
      - {'host': '/srv/podman/caddy/config', 'container': '/etc/caddy'}
      - {'host': '/srv/podman/caddy/data', 'container': '/data'}
  tags: podman_container_create
```

### 6. Enable Systemd Integration

After creating containers, enable systemd unit management. Containers must be created with `state: created`:

```yaml
- name: Enable systemd for pod or container
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_systemd_restart_pod_or_container
    target: "{{ pod_name if pod_name | d('') is truthy else container_name }}"
  tags: podman_systemd_restart_pod_or_container
```

### 7. Create Networks

Create networks for use with containers:

```yaml
- name: Create custom networks
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_network_create
    network:
      name: 'custom_network'
      subnet: '10.99.0.0/16'
      gateway: '10.99.0.1'
      iprange: '10.99.0.128/25'
  tags: podman_network_create
```

### 8. System Prune

Clean up unused resources (images, containers, volumes, networks) when disk space is low:

```yaml
- name: System prune for Podman (cleanup images and unused resources)
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_system_prune
    podman_system_prune: true
    podman_system_prune_min_free_gb: 5
    podman_system_prune_force: true  # Set to true to also remove untagged images
  tags: podman_system_prune
```

**Note**: By default, only unused (dangling) images are removed. Set `podman_system_prune_force: true` to also remove untagged images that are not in use.

### 9. Pull Container Images

```yaml
- name: Pull container images
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_image_pull
    image_name: 'docker.io/library/nginx'
    image_tag: 'latest'
  tags: podman_image_pull
```

**Remove specific images by tag:**

```yaml
- name: Remove specific outdated images
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_image_pull
    podman_rootless: true
    image_tags_to_remove:
      - 'myapp:old-version'
      - 'myapp:1.0.0'
      - 'nginx:1.19'
  tags: podman_image_pull
```

### 10. Build Container Images

```yaml
- name: Build container image
  ansible.builtin.include_role:
    name: sleif.podman
  vars:
    podman_operation: podman_image_build
    image_name: 'my_app'
    image_tag: 'v1.0'
    podman_image_copy_tmp_dir_rootless: '/tmp'  # Override temp dir if needed
  tags: podman_image_build
```

## Complete Workflow Example

Here's a complete example that combines multiple operations:

```yaml
---
- name: Setup Podman Infrastructure
  hosts: container_hosts
  user: root
  tasks:
    # Step 1: Install Podman
    - name: Install Podman
      ansible.builtin.include_role:
        name: sleif.podman
      vars:
        podman_operation: podman_install
      tags: podman_install

    # Step 2: Initialize variables for other roles
    - name: Initialize Podman environment
      ansible.builtin.include_tasks: "{{ lookup('first_found', params) }}"
      vars:
        params:
          files: sleif.podman/tasks/includes/podman_init_vars.yml
          paths: "{{ lookup('config', 'DEFAULT_ROLES_PATH') }}"
      tags: always

    # Step 3: Create pod with networks
    - name: Create application pod
      ansible.builtin.include_role:
        name: sleif.podman
      vars:
        podman_operation: podman_pod_create
        pod_name: 'production_app'
        podman_networks:
          frontend:
            name: 'frontend_network'
            subnet: '10.0.1.0/24'
            gateway: '10.0.1.1'
          backend:
            name: 'backend_network'
            subnet: '10.0.2.0/24'
            gateway: '10.0.2.1'
      tags: podman_pod_create

    # Step 4: Create containers
    - name: Create web server container
      ansible.builtin.include_role:
        name: sleif.podman
      vars:
        podman_operation: podman_container_create
        target: "{{ pod_name }}"
        container_name: 'web'
        hostname: 'web'
        podman_networks:
          frontend:
            name: 'frontend_network'
      tags: podman_container_create

    # Step 5: Enable systemd integration
    - name: Enable systemd for pod
      ansible.builtin.include_role:
        name: sleif.podman
      vars:
        podman_operation: podman_systemd_restart_pod_or_container
        target: "{{ pod_name }}"
      tags: podman_systemd_restart
```

## License

MIT

## Author Information

Created in 2023 by Sebastian Berthold  
Enhanced with socket functionality in 2025 by Jens Gecius
