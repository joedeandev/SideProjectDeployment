# Side Project Deployment Template

This repository is a template for a simple single-server multi-site deployment scheme, based on [NGINX](https://nginx.org/) and [Docker](https://docs.docker.com/compose/). This template makes it easier to manage multiple projects on a single server by setting up an extensible composition that serves most needs out of the box.

## Usage

Make sure that you have Python 3.x and Docker Compose installed. Clone the repository and run `python cmd.py docker_compose_up`. On Linux, make sure that entrypoint scripts have execution permisisons (`chmod +x ./entrypoints/nginx.sh`) See "Extending" for instructions on extending the template to include other containers.

## Containers

This template includes three pre-configured Docker containers.

### Nginx

[NGINX](https://hub.docker.com/_/nginx) is used to direct incoming web traffic to the appropriate project containers, including handling HTTP to HTTPS redirects and SSL certificates. The entrypoint script generates self-signed certificates for each domain given in the `DOMAINS` variable in the .env file (Certbot is used to generate live certificates), which allows NGINX to start before live certificates are generated.

### Certbot

[Certbot](https://hub.docker.com/r/certbot/certbot) is included and configured to automatically renew SSL certificates.

### Docker Registry

A [Docker registry container](https://hub.docker.com/_/registry) is included in this project. It serves as an easy way to manage project images, and as an example of how this template can be extended to deploy other containers. This registry requires authentication by default, and is configured to be served at any host with a .registry subdomain.

When deploying this template, be sure to configure the SSL cert paths in [registry.nginx](./nginx/registry.nginx) to match the correct hostname.

See [docker-compose.registry.yml](./compositions/docker-compose.registry.yml) and [registry.nginx](./nginx/registry.nginx) for more.

## Commands

Management commands are collected in [cmd.py](./cmd.py), which is written for Python 3.x. Python is chosen for its ease of execution, large standard library, and cross-platform support.

- `cmd.py available_commands`: Lists available commands.
- `cmd.py docker_compose_up`: Spins up a Docker composition that includes all files in the `compositions` directory.
- `cmd.py docker_compose_down`: Spins down the aforementioned composition.
- `cmd.py docker_compose_reload`: Restarts Docker composition.
- `cmd.py registry_create_password <username> <password>`: Sets the authentication for the Docker registry to the given username and password.
- `cmd.py certbot_new_domain <domain> <email>`: Gets a new certificate for the given domain, registered to the given email. Add `wildcard` to the argument list to request a wildcard certificate that covers all subdomains.

## .env

This .env file is loaded across all docker-compose files. Therefore, any environment variables referenced in these files should be included here. By default, this includes `COMPOSE_PROJECT_NAME` (to ensure that the container, network, and volume names are consistent), and [`CSR_SUBJ`](https://www.openssl.org/docs/man1.0.2/man1/openssl-req.html) and `DOMAINS`, which are used by the NGINX entrypoint to generate self-signed certs.

## Extending

New Docker images can be included in this composition by including a docker-compose file in the compositions directory (in the format `docker-compose.<project>.yml`), and an NGINX fragment in the nginx directory (in the format `<project>.nginx`). These are automatically included in the project composition when run from [cmd.py](./cmd.py).

By default, new docker-compose files inherit network settings and variables in the `.env` file. This allows NGINX to direct web traffic to specific containers.

An example NGINX configuration is as follows:

```nginx
upstream example {
	server example:8000;
}

server {
	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	server_name example.com;

	ssl_certificate /etc/nginx/ssl/live/example.com/fullchain.pem;
	ssl_certificate_key /etc/nginx/ssl/live/example.com/privkey.pem;

	location / {
		proxy_pass http://example;
	}
}
```

Note the `upstream` directive, which directs traffic to a container. This means that SSL certificates only need to be managed by the NGINX container, and sibling containers can ignore it safely. The `ssl_certificate` directives provide a path to the SSL certificates, which are generated at startup based on the `DOMAIN` variable in the .env file, and can be replaced by Certbot.

An example docker-compose file is as follows:

```yaml
volumes:
  persistent:

services:
  registry:
    container_name: ${COMPOSE_PROJECT_NAME:-deployment}_example
    image: alpine:latest
    restart: always
    volumes:
      - persistent:/data
    environment:
      VARIABLE: "${VARIABLE:-default}"
```

Environment variables should be given defaults, and defined in this repository's `.env` file (or, of course, on the server itself).

See [docker-compose.registry.yml](./compositions/docker-compose.registry.yml) and [registry.nginx](./nginx/registry.nginx) for an example of how this template can be extended.
