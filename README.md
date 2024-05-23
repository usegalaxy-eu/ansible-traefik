# Traefik Ansible Role

Role that sets up a Docker swarm and creates Traefik reverse proxy and optionally others using Geerlingguy's Docker and Pip roles.

## Requirements

On *both* the control and the remote node:
- docker
- docker-py
- rsync
(We need Docker to validate the rules files before copying them.)

On the remote node:  
dnf-packages:
  - python3-policycoreutils
  - python3-libselinux
  - policycoreutils-python-utils
  - python3-pip
python-packages:
  - docker
  - selinux

## Role Variables
See the examples and the `defaults/main.yml`
## Dependencies

This Role uses the ansible community docker collection and the already mentioned roles.
(collection: community.docker, geerlingguy.pip, geerlingguy.docker)

The dependencies should be automatically installed during role installation.

## Example
### Playbook

A basic playbook example could look like this:

```yaml
---
# Traefik test playbook
- name: Traefik
  become: yes
  hosts: all
  vars_files:
    - secret_group_vars/all.yml
    - secret_group_vars/traefik.yml
    - group_vars/all.yml
  pre_tasks:
    - name: Install ansible dependencies
      ansible.builtin.package:
        name:
          - python3-policycoreutils
          - python3-libselinux
          - policycoreutils-python-utils
          - python3-pip

    - name: Install python docker
      become: true
      ansible.builtin.pip:
        name: "{{ item }}"
        virtualenv_command: "python3 -m venv"
      loop:
        - docker
        - selinux
  roles:
    - traefik
```
### Vars file
~~~yaml
---
hostname: example.com

# This works only with dj-wasabi' telegraf role:
#telegraf_plugins_extra:
#  listen_galaxy_routes:
#    plugin: "prometheus"
#    config:
#      - urls = ["http://127.0.0.1:8082"]
#      - metric_version = 2
#      - name_override = "traefik"

# The validation works only with YAML files, no TOML, no commands in the traefik_containers section
traefik_force_validation: true # use old schema for traefik >= v3

# Rules dir defaults to files/traefik/rules
traefik_copy_rules: true

traefik_no_service_log: true # if false secrets can be revealed in the logs
traefik_manage_secrets: true

# These are created by the role if not present
traefik_log_dir: /var/log/traefik
traefik_dir: /etc/traefik

traefik_user: centos
traefik_group: centos

# using usegalaxy_eu.logrotate role
#lp_logrotate_confd:
#  - path: traefik
#    conf: |
#      {{ traefik_log_dir }}/*.log {
#        compress
#        copytruncate
#        daily
#        notifempty
#        missingok
#        rotate 1
#      }



traefik_domain: "{{ hostname }}"
# Only if making traefik accessible via tailscale
#tailscale_domain: another-animal.ts.net
#tailscale_authkey: "{{ tailscale_auth_key_usegalaxy_eu }}"

traefik_networks:
  traefik:
    internal: false
    driver: overlay
traefik_containers:
  - name: traefik
    image: traefik:v3.0
    url: "https://traefik.{{ tailscale_domain }}/dashboard/#/" # replace with traefik_domain if not using tailscale
    state: started
    restart_policy: on-failure
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    environment:
      # comment out/remove depending on what you use
      # CF_API_EMAIL_FILE: "/run/secrets/cloudflare_email"
      # CF_API_KEY_FILE: "/run/secrets/cloudflare_api_key"
      # CLOUDFLARE_DNS_API_TOKEN_FILE: "/run/secrets/cloudflare_zone_token" # more secure alternative to API Key, but was not working 5/2024
      AWS_SHARED_CREDENTIALS_FILE: "/run/secrets/aws_shared_credentials"
    networks:
      - traefik
    command:
      - --ping
      - --ping.entryPoint=websecure
      - --api=true
      - "--providers.swarm.exposedByDefault=false"
      - "--providers.swarm.endpoint=unix:///var/run/docker.sock"
      #- "--providers.docker.network={{ traefik_networks[0]['name'] }}"
      - --api.dashboard=true
      - --api.insecure=false
      - --global.sendAnonymousUsage=false
      - --global.checkNewVersion=false
      - --entryPoints.web.address=:80
      - --entryPoints.websecure.address=:443
      - --entryPoints.metrics.address=:8082
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      # set certresolver for your domain
      - "--entryPoints.websecure.http.tls.domains[0].main={{ traefik_domain }}"
      - --entryPoints.https.http.tls.certresolver=route53 # change to 'cloudflare' if needed
      - "--entryPoints.websecure.http.tls.domains[0].sans=*.{{ traefik_domain }}"

      # Prometheus Metrics
      - --metrics.prometheus=true
      - --metrics.prometheus.buckets=0.1,0.3,1.2,5.0
      - --metrics.prometheus.addEntryPointsLabels=true
      - --metrics.prometheus.addrouterslabels=true
      - --metrics.prometheus.addServicesLabels=true
      - --metrics.prometheus.entryPoint=metrics
      - --metrics.prometheus.headerlabels.origin=Origin

      # Influx Metrics - Deprecated in v3
      # - --metrics.influxdb=true
      # - --metrics.influxdb.pushinterval=10s
      # - --metrics.influxdb.address={{ influxdb.url }}
      # - --metrics.influxdb.protocol=http
      # - --metrics.influxdb.database={{ influxdb.node.database }}
      # - --metrics.influxdb.username={{ influxdb.node.username }}
      # - --metrics.influxdb.password={{ influxdb.node.password }}

      # Traefik can watch rules files, no need to restart!
      # Make sure you validate them, if there is one error in a file, 
      # Traefik will remove all it's entities; like routers, services...
      # So it is wise to keep things separated
      - "--providers.file.directory=/rules"
      - "--providers.file.watch=true"

      ## DNS Challenge (full list of providers can be found in Traefik's docs)
      - --certificatesResolvers.ts.tailscale=true
      - --certificatesResolvers.route53.acme.storage=/letsencrypt/acme.json
      - --certificatesResolvers.route53.acme.dnsChallenge.provider=route53
        # - --certificatesResolvers.dns-cloudflare.acme.storage=/letsencrypt/acme.json
        # - --certificatesResolvers.dns-cloudflare.acme.dnsChallenge.provider=cloudflare
        # - --certificatesResolvers.dns-cloudflare.acme.dnsChallenge.resolvers=1.1.1.1:53,1.0.0.1:53
        # - --certificatesresolvers.dns-cloudflare.acme.caserver=https://acme-v02.api.letsencrypt.org/directory

      # Enable Logging
      - "--log=true"
      - --log.filePath={{ traefik_log_dir }}/traefik.log
      - "--log.level=DEBUG" # DEBUG, PANIC, FATAL, ERROR, WARN, INFO
      #- "--serversTransport.insecureSkipVerify=true"
      - --accesslog=true
      - --accesslog.filepath={{ traefik_log_dir }}/access.log
    labels:
      # The "labels" section is read by Traefik and sets up the routers, middlewares and services for the container
      "traefik.enable": "true"
      #"traefik.http.routers.traefik.rule": "Host(`{{ traefik_domain }}`) || Host(`www.{{ traefik_domain }}`)"
      "traefik.http.routers.traefik.rule": "Host(`traefik.{{ tailscale_domain }}`)"
      "traefik.http.routers.traefik.entrypoints": "websecure,web"
      # Remove the tls labels if not using tailscale, it then gets terminated by the default certresolver (above)
      "traefik.http.routers.traefik.tls": "true"
      "traefik.http.routers.traefik.tls.certresolver": "ts"
      "traefik.http.routers.traefik.tls.domains[0].main": "traefik.{{ tailscale_domain }}"
      "traefik.http.routers.traefik.service": "api@internal"
      # This one is needed for docker swarm
      "traefik.http.services.dummy-svc.loadbalancer.server.port": "9999"

    # For SELinux in enforcing mode
    security_opts:
      - "label:type:container_runtime_t"

    mode: global

    mounts:
      - source: "/var/run/docker.sock"
        target: "/var/run/docker.sock"
        type: bind
        readonly: true
      - source: "/var/run/tailscale"
        target: "/var/run/tailscale"
        type: bind
        readonly: true
      - source: "/etc/traefik/rules/"
        target: "/rules"
        type: bind
        readonly: true
      - source: "/etc/traefik/acme.json"
        target: "/letsencrypt/acme.json"
        type: bind
        readonly: false
      - source: "{{ traefik_log_dir }}/"
        target: "{{ traefik_log_dir }}"
        type: bind
        readonly: false

    publish:
      - target_port: 80
        published_port: 80
        protocol: tcp
      - target_port: 443
        published_port: 443
        protocol: tcp
      - target_port: 8080
        published_port: 8080
        protocol: tcp
      - target_port: 8082
        published_port: 8082
        protocol: tcp

    secrets:
      - secret_name: aws_shared_credentials
~~~
### Ansible-Vault for AWS or Cloudflare
~~~yaml
---
traefik_docker_secrets:
  cloudflare_api_key: abcdefg12345678hijklmnop
  cloudflare_email: ADMIN@EXAMPLE.COM
# Yes you can get entire files with linebreaks into Docker secrets
  aws_shared_credentials: |
    [default]
    aws_access_key_id=ABCDEFGHIJKLMNOPQRSTUVWXYZ
    aws_secret_access_key=ABCDEFGHIJKLMNOPQRSTUVWXYZ
~~~
### Rules file
Located in `files/traefik/rules`
#### Service
Create a service named `galaxy`, that is loadbalancing the workload between two bare metal nodes.
~~~yaml
http:
  services:
    galaxy:
      loadBalancer:
        passHostHeader: true
        servers:
          - url: "https://sn06.galaxyproject.eu"
          - url: "https://sn07.galaxyproject.eu"
~~~
Create a middleware named `secure-headers` that takes care of secure headers.
~~~yaml
http:
  middlewares:
    secure-headers:
      headers:
        accessControlMaxAge: 100
        hostsProxyHeaders:
          - "X-Forwarded-Host"
        stsSeconds: 63072000
        stsIncludeSubdomains: true
        stsPreload: true
        forceSTSHeader: true
        contentTypeNosniff: true
        browserXssFilter: true
        referrerPolicy: "same-origin"
        permissionsPolicy: "camera=(), microphone=(), geolocation=(), payment=(), usb=(), vr=()"
        customResponseHeaders:
          X-Robots-Tag: "none,noarchive,nosnippet,notranslate,noimageindex,"
          server: ""
~~~
Create a router for your main domain. It uses the default tls, that we defined in the vars and the middleware `secure-headers` and routes the treafik to the service `galaxy`.
~~~yaml
http:
  routers:
    sn06-rtr:
      rule: "Host(`example.com`)"
      service: "galaxy"
      tls: {}
      middlewares:
        - secure-headers
~~~
This is a template that automatically creates a router for each specified subdomain, with
1. A routing rule
2. A service `sn07` attached
3. Middleware `secure-headers` attached
3. TLS via certResolver `route53`, with automatic wildcard certs for the subdomain and the interactive tool entrypoint

~~~yaml
# files/traefik/rules/template-subdomains.yml
{{define "subdomain"}}
    prox-{{ . }}:
      rule: Host(`{{ . }}.example.com`)
      middlewares:
        - secure-headers
      service: sn07
      tls:
        certResolver: "route53"
        domains:
          - main: "{{ . }}.example.com"
            sans:
              - "*.ep.interactivetool.{{ . }}.example.com"
              - "*.{{ . }}.example.com"
{{end}}

http:
  routers:
{{template "subdomain" "live"}}
{{template "subdomain" "imaging"}}
 ...
~~~
### How to set IAM permissions in AWS
1. Create a User, name it e.g. 'treafik'
2. Create a new Policy and add the new user
3. Add this to the policy, so traefik can create and change TXT records with '__acme_challenge.' prefix
~~~json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "route53:GetChange",
            "Resource": "arn:aws:route53:::change/*"
        },
        {
            "Effect": "Allow",
            "Action": "route53:ListHostedZonesByName",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/ABCDEFGDIESEZONETUTNICHTWEH"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/ABCDEFGDIESEZONETUTNICHTWEH"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "route53:ChangeResourceRecordSetsRecordTypes": [
                        "TXT"
                    ]
                },
                "ForAllValues:StringLike": {
                    "route53:ChangeResourceRecordSetsNormalizedRecordNames": "_acme-challenge.*"
                }
            }
        }
    ]
}
~~~

## Handlers

Handlers are checking if a rollback happend and if the `url` defined for each container is reachable.

## Tests

Are currently not working, because of docker-compose inside docker problems. Contributions are welcome. The role is tested manually.

## License

CC-BY-4.0
