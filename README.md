# Traefik Ansible Role

Role that starts a Traefik2 reverse proxy using Geerlingguy's Docker and Pip roles.

## Requirements

none

## Role Variables

The role uses very few variables:

- **traefik_user** specifies the user to own the copied files and to be added to docker group
- **traefik_group** specifies the group to own the copied files
- **traefik_dir** the root path in which all files and the role directory go
- **traefik_compose_file** in case you want a different name
- **traefik_network_name** the name of created docker network

## Dependencies

This Role uses the ansible community docker collection and the already mentioned roles.
(collection: community.docker, geerlingguy.pip, geerlingguy.docker)

The dependencies should be automatically installed during role installation.

## Example Playbook

A basic playbook example could look like this:

```yaml
---
# Traefik test playbook
- name: Traefik
  become: yes
  hosts: all
  vars:
    traefik_user: rocky
    traefik_group: rocky
  roles:
    - traefik
```

Don't forget to add a files directory on playbook level, where you add the **docker-compose.yml** and the **role/** directory, in which you could add role files.

## Handlers

The docker-compose up command should is executed through a handler, each time the compose file changes.
Because traefik listens to rules file changes automatically this is not needed there.

## Tests

Are currently not working, because of docker-compose inside docker problems. Contributions are welcome. The role is tested manually.

## License

CC-BY-4.0
