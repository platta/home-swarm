# Portainer Stack

This stack stands up the Portainer server and the Portainer agent for a swarm
cluster. The server can bounce between hosts in the swarm, but we need one
instance of the agent on each host, so it is declared global.

## Setup

### 1. Deploy the stack

Since nothing else has been deployed yet, we're still working at the command
line. Assuming you've cloned this repository to your server or at least copied
the `docker-compose.yml` file for the Portainer stack there, go to the folder
where that file is and use the following command to create the stack.

```console
docker stack deploy -c docker-compose.yml portainer
```

### 2. Create the admin user

Once the stack is up and running, navigate to <http://yourhost:9000> to access the
portainer UI. It will prompt you to create a password for the admin user. You
can uncheck the anonymous statistics checkbox if you'd like.

### 3. Add the swarm environment

Now that you're logged into Portainer, you'll need to add your swarm environment
for management. Follow these steps:

- Click on "Add Environment"
- Select "Docker Swarm"
- Click on "Start Wizard"
- Select "Agent" (should be selected by default)
- Provide a name for the environment (anything is fine)
- Use `tasks.portainer_agent:9001` as the environment address
- Click "Connect"
- Click "Close"

Now that your environment is added, you can use Portainer to create additional
stacks.
