# Home Swarm

Self-host your own services from home using Docker Swarm.

## About

This repository contains multiple folders, each with a number of files related
to deploying a Docker stack. The setup these stacks will produce is
a single node Docker Swarm with the following services:

- Portainer, to managed stacks and containers
- NGINX, to act as a reverse proxy so you can host multiple services from the
  same address using different DNS names.
- Certbot, to retrieve and automatically renew free SSL certificates to protect
  your services' traffic.
- Authelia, to provide 2FA to your self-hosted services and protect your home
  network.
- Grafana (with Prometheus, Promtail, and Loki) to provide monitoring for your
  node(s) and services.

## Setup

Each of the stacks you'll be standing up has its own folder in the repository
along with its own README.md file. The majority of the instructions will be
there.

But!

There are a few things you'll have to do first.

### 1. Set up Docker and Swarm

The assumption is that you're going to be running this on some flavor of Linux.
Check out [the instructions](https://docs.docker.com/engine/install/) to get it
installed on your distro. I highly recommend following the steps to [manage Docker as a non-root user](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)
or else things can get really annoying very quickly.

Once Docker is up and running, you'll need to [create a swarm](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/).
Note that the `--advertise-addr` argument isn't really required, especially for
a home setup.

### 2. Create the shared_net network

The `shared_net` network is used by multiple stacks to expose specific
containers to NGINX for reverse proxying. Since Portainer needs to be on that
network and Portainer gets stood up before NGINX, we need to create that
network before anything else. That means we've got to use the command line.

```console
docker network create --scope=swarm -d overlay shared_net
```

This creates a network called `shared_net` as an overlay network across your
swarm. That means it is a virtual network that will extend to all nodes in your
swarm so any container on any node that is attached to that network will be able
to communicate with any other containers on any note in the swarm that are also
attached to it. Swarm is great at hiding this complexity and automatically
linking and load balancing containers across nodes.

### 3. Stand up the stacks

Now you're ready to deploy the stacks that will get your swarm up and running.
Follow the instructions in each subfolder, and deploy the stacks in the order
below.

1. Portainer
2. NGINX
3. Monitoring

Once you have those stacks deployed, you have the foundation you need to deploy,
expose, and monitor anything else you want!

## Swarm Notes

### On single node swarms

Why are we creating a single node swarm? Isn't swarm for managing Docker
clusters spanning multiple physical hosts? It is! But, running even a single
node in swarm mode gives us a few extra tools that are super useful.

#### Configs

Configs are resources you define at the swarm level that can be injected into
your services. They are a perfect way to get configuration files into your
environment without having to create a volume. One thing to know about Configs
is that they are read-only once created, so if you need to update something you
have to create a new Config, update your stack to reference the new Config, and
redeploy your stack.

I like to use timestamps in the format `_YYYY-MM-DD_HHmm` at the end of all my
Config names so I can easily sort through old ones (if I haven't deleted them).

### Secrets

Secrets are a lot like Configs, except they are for (you guessed it) secrets.
Some stacks require you to specify a JWT key or other key value to secure data
or communication. Secrets are read-only, just like Configs, but you cannot view
their contents once created. You can inject a Secret into a service just like a
Config, and it will appear as a file inside the container. Many services allow
you to specify a file location as the source for secret values, and this is
exactly how you can leverage that option.

### On storage

You can technically expand this setup to span multiple nodes. The service
definitions in the stacks are designed to support that, but there is one caveat.
Swarm doesn't manage storage in any way. That means if you're going to move to a
multi-node swarm you're going to need to either introduce a SAN or some kind of
local storage sync between your nodes. Otherwise, when services move from node
to node their corresponding storage volumes won't follow them.

This is part of the reason Configs and Secrets are great, because they _do_
replicate across nodes.

## Additional Notes

- Swarm names services (and their DNS names) using the format `stack_service`,
  so many of the pieces of the compose templates and configuration files are
  dependent on using the containing folder name for each stack you deploy. You
  can technically use any name you want, but you'll need to update all
  references to those names.
- In the same way, swarm will name other resources `stack_resource` as well.
  This is why each stack creates a network called `net`. It just makes it easier
  to read when looking at the list of networks in portainer.
