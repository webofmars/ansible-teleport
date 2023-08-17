# webofmars/teleport

A role for deploying and configuring [teleport](https://goteleport.com) and extensions on unix hosts using [Ansible](http://www.ansible.com/).

## Requirements

Supported targets:

- Ubuntu 22.04 "Jammy"
- Debian 11 "Bullseye"
- Debian 12 "Bookworm"

## Role Variables

This roles comes preloaded with almost every available default. You can override each one in your hosts/group vars, in your inventory, or in your play. See the annotated defaults in `defaults/main.yml` for help in configuration. All provided variables start with `teleport__`.

- `teleport__version: 8` - Major Version / branch of the binary to install and hold, default install repo version 8. Available now: 8, 9, 10, 11.
- `teleport__agent: false` - Configure and Enable the teleport agent software.
- `teleport__bind_addr: 0.0.0.0` - Bind address used to default all other bind address configuration
- `teleport__nodename` - Name the teleport agent reports to its connected proxy. If undefined, no nodename will be configured and Teleport will default to the machine's hostname.
- `teleport__diag: false` - Enable teleport HTTP monitoring endpoint.
- `teleport__diag_addr: "127.0.0.1"` - Bind address for HTTP monitoring endpoint.
- `teleport__diag_port: 3000` - Port to bind for HTTP monitoring endpoint.
- `teleport__node: false` - Configure and Enable teleport node role.
- `teleport__node_token: ""` - Token used to join the proxy.
- `teleport__node_server: ""` - Proxy server url.
- `teleport__proxy: false` - Enable the proxy mode in teleport.
- `teleport__proxy_public_addr: ""` - Public address the proxy expose.
- `teleport__proxy_acme: false` - Enable ACME protocol for public certificate. When disabled, Teleport will look for certificate in `/etc/letsencrypt/live/{{ teleport__proxy_public_addr }}`
- `teleport__proxy_acme_email: ""` - Email for the ACME request.
- `teleport__auth: false` - Enable teleport auth role.
- `teleport__auth_cluster_name: ""` - Teleport auth cluster name.
- `teleport__auth_u2f: false` - Enable U2F (old-style non-webauthn configuration)
- `teleport__auth_addr: {{ teleport__bind_addr }}` - Bind address for auth teleport service
- `teleport__auth_port: 3025` - Port to bind for auth teleport service
- `teleport__ssh_addr: {{ teleport__bind_addr }}` - Bind address for ssh teleport service
- `teleport__ssh_port: 3022` - Port to bind for ssh teleport service
- `teleport__ssh: false` - Enable teleport ssh module.
- `teleport__ssh_labels: ''` - Add labels to the ssh module (yaml format).
- `teleport__app: false` - Enable teleport app module
- `teleport_applications: []` - List of applications, defined as a dict with the following keys:
  - `name` - Name of the application
  - `uri` - URI to reverse-proxify
  - `skip_verify: false` - Whether or not to skip certificate verification on the target URI
- `teleport__db: false` - Enable teleport db module
- `teleport_databases: []` - List of databases, defined as a dict with the following keys:
  - `name` - Name of the database instance in teleport
  - `protocol` - Name of the protocol (like mysql / postgresql)
  - `uri` - URI of the database from the node (have to match the certificate)
  - `static_labels` - List of labels to be applied to the database
- `teleport__web_addr: {{ teleport__bind_addr }}` - Bind address for web teleport service
- `teleport__web_port: 443` - Port to bind for web teleport service
- `teleport__tunnel_addr: {{ teleport__bind_addr }}` - Bind address for tunnel service
- `teleport__tunnel_port: 3024` - Port to bind for tunnel teleport service
- `teleport__binary_compat: false` - If true will deploy a binary version beside the package with more glibc compatibility. (Automatically done on debian pre buster (10) releases)
- `teleport__install_repo: true` - Set to false if you want to prevent repo installation (usefull for airgap environnement - install manually then use this role to configure everything)

## Dependencies

- None

## Usage

Use Ansible galaxy requirements.yml

```yaml
# teleport from webofmars
# private role
- src: git://git@github.com:webofmars/ansible-teleport.git
  name: webofmars.teleport
```

And add it to your play's roles:

```yaml
# Node example
- hosts: all
  roles:
    - role webofmars.teleport:
        teleport__agent: true
        teleport__version: 13
        teleport__nodename: "test.node"
        teleport__node: true
        teleport__node_token: "gjlksfdjglkfsdjlkgfds9423"
        teleport__node_server: "https://teleport.example.com:3025"
        teleport__ssh: true
        teleport__ssh_labels:
          tenant: toto.com
```

```yaml
# Proxy example
- hosts: all
  roles:
    - role webofmars.teleport:
        teleport__agent: true
        teleport__version: 13
        teleport__nodename: "teleport.proxy"
        teleport__proxy: true
        teleport__proxy_public_addr: "teleport.example.com"
        teleport__proxy_acme: false
        teleport__proxy_acme_email: "contact@example.com"
        teleport__auth: true
        teleport__auth_cluster_name: "teleport.example.com"
        teleport__ssh: true
        teleport__ssh_labels:
          tenant: exampledotcom
```

```yaml
# Full example
---

# use certbot + apache to generate teleport server cert
- hosts: teleport_servers
  tags: teleport_servers_certs
  pre_tasks:
    - name: Install apache2
      apt:
        name: apache2
        state: present
    - name: Start apache2 (once)
      service:
        name: apache2
        state: started
        enabled: no
  roles:
    - role: systemli.letsencrypt
      vars:
        letsencrypt_account_email: "contact+exampledotcom@example.com"
        letsencrypt_http_auth: apache
        letsencrypt_webroot_path: /var/www/html
        letsencrypt_cert:
          name: "teleport.example.com"
          domains:
            - "teleport.example.com"
            - "nginx.teleport.example.com"
          challenge: http
          services:
            - apache2
  tasks:
    - name: Stop apache2
      service:
        name: apache2
        state: stopped
        enabled: no

# install teleport server
- hosts: teleport_servers
  tags: teleport_servers_teleport
  roles:
    - role: webofmars.teleport
      teleport__agent: true
      teleport__nodename: "teleport.example.com"
      teleport__proxy: true
      teleport__proxy_public_addr: "teleport.example.com"
      teleport__proxy_acme: true
      teleport__proxy_acme_email: "contact@example.com"
      teleport__node: false
      teleport__auth: true
      teleport__auth_cluster_name: "teleport.example.com"
      teleport__ssh: true
      teleport__ssh_labels:
        tenant: exampledotcom
        role: teleport

- hosts: demo_nodes
  tags: demo_nodes
  tasks:
  - name: install fake apps
    apt:
      name: "{{ item }}"
      state: present
    with_items:
    - nginx
    - mariadb-server
    - mongodb-org # need to follow https://www.fosslinux.com/65442/how-to-install-mongodb-on-debian-11.htm before

- hosts: teleport_nodes
  tags: teleport_nodes
  roles:
    - role: webofmars.teleport
      teleport__agent: true
      teleport__nodename: "{{ inventory_hostname }}"
      teleport__node: true
      teleport__node_token: "{{ teleport_node_token }}"
      teleport__node_server: "teleport.example.com:443"
      teleport__ssh: true
      teleport__ssh_labels:
        tenant: exampledotcom
        role: node
        teleport_roles: "ssh,mariadb,mongodb,web"
      teleport__app: true
      teleport_applications:
        - name: nginx
          protocol: tcp
          uri: "http://localhost"
          labels:
            tenant: exampledotcom
            role: web
      teleport__db: true
      teleport_databases:
        - name: demo-mariadb
          protocol: mysql
          uri: "127.0.0.1:3306"
          static_labels:
            tenant: exampledotcom
            role: mariadb
          dynamic_labels:
            - name: "hostname"
              command: ["hostname"]
              period: 1m0s
        - name: demo-mongodb
          protocol: mongodb
          uri: "mongodb://127.0.0.1:27017"
          static_labels:
            tenant: exampledotcom
            role: mongodb
          dynamic_labels:
            - name: "hostname"
              command: ["hostname"]
              period: 1m0s
```

## License

- MIT
