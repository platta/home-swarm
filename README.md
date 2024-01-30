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

Note - You can technically expand this setup to span multiple nodes. The service definitions in the stacks are designed to support that, but there is one caveat. Swarm doesn't manage storage in any way. That means if you're going to move to a multi-node swarm you're going to need to either introduce a SAN or some kind of
local storage sync between your nodes. Otherwise, when services move from node
to node their corresponding storage volumes won't follow them.

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
