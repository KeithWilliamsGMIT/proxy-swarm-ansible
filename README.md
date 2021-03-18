# Proxy Swarm Ansible

This repository shows how to setup a reverse proxy for services running in Docker Swarm using [Traefik](https://docs.traefik.io/). Ansible is used to automate the various tasks. This repository can be used to run the proxy locally using self-signed certs or on a cloud environment. DigitOcean is used as an example.

## Getting Started

The first step to getting started is to clone the repository:

```bash
git clone https://github.com/KeithWilliamsGMIT/proxy-swarm-ansible.git
```

Next, install Ansible and then we can use two existing Ansible roles to install Docker and initialise a Docker Swarm. These roles are defined in the `requirements.yml` file and will be installed to `~/.ansible/roles/` by default using the below commands:

```bash
cd playbooks/requirements
ansible-galaxy install -r requirements.yml
```

We can install [Docker](https://docs.docker.com/install/) on all the hosts now using the `install-docker.yml` playbook as shown below:

```bash
cd playbooks
ansible-playbook -i ./inventories/localhost install-docker.yml
```

Similarly, to initialise a Docker Swarm between the hosts we can run the `initialize-swarm.yml` playbook. Note that we set extra variables to override some defaults in the playbook. We want to skip everything except the actual initialisation of Docker Swarm.

```bash
ansible-playbook -i ./inventories/localhost initialize-swarm.yml --extra-vars="{'skip_engine': 'True', 'skip_group': 'True', 'skip_docker_py': 'True'}"
```

Finally, we can deploy the stack containing the proxy service to Docker Swarm using the `deploy-proxy.yml` playbook.

```bash
ansible-playbook -i ./inventories/localhost deploy-proxy.yml --ask-vault-pass --ask-become-pass
```

The `--ask-vault-pass` flag will prompt you for the vault password to access sensitive data. A default password used in this case, for demostraion purposes, is `password`.

## Running the proxy locally

To run the proxy locally with a domain name you will need to start a local DNS server. For example, `dnsmasq` can be used with Ubuntu. The first step to configuring `dnsmasq` on Ubuntu is to add the line `dns=dnsmasq` to the `[main]` section of the `/etc/NetworkManager/NetworkManager.conf` file so that is looks like the following:

```bash
[main]
dns=dnsmasq
```

Next, remove the `/etc/resolv.conf` file and instead create a link from the NetworkManager `resolv.conf` configuration file.

``` bash
sudo rm /etc/resolv.conf
sudo ln -s /var/run/NetworkManager/resolv.conf /etc/resolv.conf
```

Then, create a new configuration file for the domain you want to use locally and point it to localhost. By default, this repository uses `example.local` so we will configure this domain.

```bash
echo 'address=/.example.local/127.0.0.1' | sudo tee /etc/NetworkManager/dnsmasq.d/example.local.conf
```

Finally, restart the NetworkManager so that the changes will take effect.

```bash
sudo systemctl reload NetworkManager
```

To test that the DNS server will direct any request for `example.local`, or any of it's subdomains, to localhost, use the `dig` command as shown below:

```bash
ubuntu:~$ dig example.local +short
127.0.0.1
```

To run the proxy with HTTPS you will also need to generate self-signed SSL certs. These are provided in the repository under `playbooks\roles\deploy-proxy\files`. If these have expired or you wish to create your own you can generate new ones using the following command:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout local.key -out local.crt
```

## Does it work?

Once you followed the above steps we can check if the proxy service is running as expected on the docker manager.

```bash
ubuntu:~/proxy-swarm-ansible/playbooks$ sudo docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
dj5m4yokvqas        proxy_traefik       global              1/1                 traefik:latest      *:80->80/tcp, *:443->443/tcp
```

We can now go to the Traefik dashboard by navigating to [https://traefik.proxy.example.local/](https://traefik.proxy.example.local/). There is basic authentication configured for the dashboard by default. Login using the username `admin` and password `password`.

## Further configuration

### Configuring other services

Traefik uses labels in Docker Swarm to configure other services. For example, to access the Traefik service at `traefik.example.com` we can use the following labels.

```bash
deploy:
    labels:
    - "traefik.port=8080"
    - "traefik.backend=traefik"
    - "traefik.frontend.rule=Host:traefik.proxy.example.com"
    - "traefik.docker.network=proxy_network"
```

### Ansible variables

There are also a number of Ansible variables that can be overridden. These can be found in the table below:

| Variable Name | Description | Default Value |
|---------------|-------------|---------------|
| temporary_directory | A path to a directory where temporary files such as the compose file can be written to | "/tmp" |
| volume_directory | A path to a directory where service data can be persisted | "/tmp" |
| docker_network_name | The name of the new Docker overlay network | "proxy_network" |
| traefik_version | The tag of the Traefik Docker image to use | v1.7.20 |
| traefik_user | The username of the user to log into Traefik | admin |
| traefik_email_address | The email address used for the Traefik service | "admin@domain.com" |
| domain_name | The domain name used to access the services | example.local |
| is_using_local_ssl_certs | Boolean value used to determine whether to use SSL certs generated by LetsEncrypt or self-signed certs | true (depends on domain name) |
| is_using_acme_staging | Boolean value used to determine whether to use the LetsEncrypt staging or production environment | false (depends on domain name) |
| is_traefik_metrics_enabled | Boolean value used to determine whether to expose Prometheus metrics from the `/metrics` endpoint. | true |
| is_traefik_tracing_enabled | Boolean value used to determine whether to enable tracing using Jaeger. See [tracing-swarm-ansible](https://github.com/KeithWilliamsGMIT/tracing-swarm-ansible) for an example on how to run Jaeger using Docker Swarm. | true |

## Using this role

To use this role add the following to your `requirements.yml` file:

```
- src: https://github.com/KeithWilliamsGMIT/proxy-swarm-ansible.git
  version: master
  name: deploy-proxy
```

## Troubleshooting

Here are a few tips you can try to troubleshoot any issues you may encounter:

+ Make sure that the Traefik service is running as expected.
+ Use the `dig` command, or similar, to check that your chosen domain is resolved to one of the swarm manager nodes.

## Contributing

Any contribution to this repository is appreciated, whether it is a pull request, bug report, feature request, feedback or even starring the repository. Some potential areas that need further refinement are:

+ Hardening of role
+ Publishing role to Ansible Galaxy
+ Documentation
+ Upgrade to Traefik v2

## Conclusion

This repository demonstates how to use the Traefik reverse proxy in Docker Swarm. This includes configuring other services to use this proxy and enabling HTTPS on all services either with letsEncrypt or self-signed certs. Finally, running all of these commands manually everytime we need to deploy the reverse proxy would be time consuming and so Ansible is quite useful in automating this process.
