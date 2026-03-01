# sleif.podman

This role installs and manages Podman.

The role supports different `podman_operations`.

```yaml
podman_operations:
  - podman_container_create
  - podman_image_build
  - podman_image_pull
  - podman_init_vars
  - podman_install
  - podman_network_create
  - podman_socket_create
  - podman_pod_create
  - podman_systemd_restart_pod_or_container
```

- podman_install: installs Podman and prepares rootfull and rootless operations
- podman_init_vars: provides Podman environments for other roles; it will provide:
  - _container_storage_dir_base_local
  - _container_storage_dir_base
  - _group, _owner
  - _systemd_scope
  - _systemd_service_files_dir
  - _xdg_runtime_dir
- podman_systemd_restart_pod_or_container: provides systemd unit file management; to ensure that Podman containers will work smoothly with systemd, it is required to just create `state: created` the containers with `containers.podman.podman_container` and call `podman_systemd_restart_pod_or_container` after the pod creation
- podman_pod_create: creates a Podman pod to be used from other roles
- podman_network_create: creates Podman networks as defined as dict in podman_networks automagically while creating containers or pod (if required). Of course, in proper context (rootful/rootless). Aside from podman itself defined networks no networks are created when installing podman.
- podman_socket_create: creates a socket (systemd-unit) for a particular container (e.g. caddyserver to do networking rootless on host ports without networking involved)

- ```podman_log_driver``` is available to define the podman logdriver for a rootless installation. It has ```k8s-file``` as default.
- ```podman_image_copy_tmp_dir_root``` and ```podman_image_copy_tmp_dir_rootless``` can define another directory to circumvent size constraints on /var/tmp (rootless) for image building and image updates.
## Requirements

None

## Role Variables

- podman_operation

## Dependencies

None

## Example Playbook

- install Podman:

```yaml
- name: VM caddy.example.com
  hosts: "caddy.example.com"
  user: root

  roles:
    - {role: sleif.podman, tags: "podman_role",
       podman_operation: "podman_install"}
```

- Calls from inside other roles:
  - Initialize the Podman environment

  ```yaml
  # the next task will return:
  # - _container_storage_dir_base_local
  # - _container_storage_dir_base
  # - _group, _owner
  # - _systemd_scope
  # - _systemd_service_files_dir
  # - _xdg_runtime_dir
  - name: Include podman_init_vars.yml from sleif.podman
    ansible.builtin.include_tasks: "{{ lookup('first_found', params) }}"
    vars:
      params:
        files: sleif.podman/tasks/includes/podman_init_vars.yml
        paths: "{{ lookup('config', 'DEFAULT_ROLES_PATH') }}"
    tags: always
  ```

  - Create a Podman pod:

  ```yaml
  - name: Include role sleif.podman podman_operation podman_pod_create
    ansible.builtin.include_role:
      name: sleif.podman
      apply:
        tags:
          - podman_pod_create
    vars:
      podman_operation: podman_pod_create
      podman_rootless: true
      podman_networks:
        - podman_network_name: 'podman_net1'
          podman_network_subnet: '10.0.0.0/24'
          podman_network_gateway: '10.0.0.1'
          podman_network_iprange: '10.0.0.128/25'
        - podman_network_name: 'podman_net2'
          podman_network_subnet: '10.0.1.0/24'
          podman_network_gateway: '10.0.1.1'
          podman_network_iprange: '10.0.1.128/25'
    tags: always
  ```

  - Trigger systemd unit file creation and service enable/start

  ```yaml
  - name: Include role sleif.podman podman_operation podman_systemd_restart_pod_or_container
    ansible.builtin.include_role:
      name: sleif.podman
      apply:
        tags:
          - podman_systemd_restart_pod_or_container
    vars:
      podman_operation: podman_systemd_restart_pod_or_container
      target: "{{ pod_name if pod_name | d('') is truthy else container_name }}"
    tags: always
  ```

  - Create a complete container

  ```yaml
  - name: Include role sleif.podman podman_operation podman_container_create
    ansible.builtin.include_role:
      name: sleif.podman
      apply:
        tags:
          - podman_container_create
    vars:
      podman_operation: podman_container_create
      target: "{{ pod_name if pod_name | d('') is truthy else container_name }}"
      container_name: 'foo'
      hostname: 'foo-host'
      podman_networks:
        - podman_network_name: 'podman_net1'
          podman_network_subnet: '10.9.0.0/24'
          podman_network_gateway: '10.9.0.1'
          podman_network_iprange: '10.9.0.128/25'
        - podman_network_name: 'podman_net2'
          podman_network_subnet: '10.10.0.0/24'
          podman_network_gateway: '10.10.0.1'
          podman_network_iprange: '10.10.0.128/25'
      podman_sockets:
        container: "{{ container_name }}"
        sockets:
          - "ListenStream=[::]:80"    # in caddy: fd/3
          - "ListenStream=[::]:443"   # in caddy: fd/4
          - "ListenDatagram=[::]:443" # in caddy: fdgram/5
          - "ListenStream=[::1]:2019"  # in caddy: fd/6
      volumes:
        - {'host': '/srv/podman/container_data/{{ container_name }}/data', 'container:' '/data'}
      secrets:
        - {'name': 'webpassword', 'data': '{{ password }}'}
    tags: always
  ```

## License

MIT

## Author Information

Created in 2023 by Sebastian Berthold
(network, sockets enhanced: Jens Gecius 2025)
