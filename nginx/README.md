# NGINX Stack

This stack includes three services that all work together.

- NGINX performs actual reverse proxy operations.
- Certbot requests and renews free SSL certificates from
  [Let's Encrypt](https://letsencrypt.org).
- Authelia (and Redis) puts an authentication barrier in front of all reverse
  proxied services with support for MFA.

The assumption is that you've got a domain (e.g. yoursite.com) and each service
you deploy will be hosted at a subdomain (e.g. service.yoursite.com). This stack
will allow you to receive traffic on ports 80 and 443 and reverse proxy that
traffic to the appropriate service in your swarm based on the DNS name of the
incoming request.

## Setup

### 1. Create Configs

The compose template for this stack references a number of Configs and Secrets.
You'll need to create those in Portainer UI and then specify the correct names
in the template when you deploy the stack.

For example, at the top of the stack template, there's a line:

```yaml
x-nginx-conf: &nginx-conf nginx_nginx.conf_YYYY-MM-DD_HHmm
```

If you named your Config `nginx_nginx.conf_version1`, you'd want to change the
line in the stack template to:

```yaml
x-nginx-conf: &nginx-conf nginx_nginx.conf_version1
```

#### nginx_nginx.conf

This is the most important Config. It defines all the services to reverse proxy
and how to redirect incoming traffic.

Initially, you don't need to modify the template file. As you deploy services,
you will need to update the configuration multiple times (you'll see this with
Authelia while deploying this stack).

#### nginx_nginx_snippets_auth.conf

This snippet is referenced by the main configuration file. It defines settings
for routing traffic to Authelia for authentication.

The only modification you need to make to this file is to insert the actual auth
URL you intend to use (i.e. change `auth.yoursite.com` at the bottom of the file
to your actual auth URL).

#### nginx_nginx_snippets_proxysettings.conf

This snippet is referenced by the main configuration file. It defines proxy
settings that ensure the reverse proxy action is configured correctly.

You do not need to modify the template file.

#### nginx_nginx_snippets_authelia.conf

This snippet is referenced by the main configuration file. It defines a location
under each reverse proxied service that Authelia uses to test the user's
authentication.

You do not need to modify the template file.

#### nginx_authelia_configuration.yml

This is the configuration file for Authelia. As you add services to your site,
you will need to update this file with settings for each subdomain. There are
examples in comments in the template file.

You will need to change the `session.domain` property and the
`access_control.rules` entry for `auth.yoursite.com` from `yoursite.com` to
your actual domain. Beyond that, you won't need to modify this file until you
deploy a new service.

### 2. Create Secrets

Secrets are used in the stack template the same way Configs are. Create the
secrets in Portainer, and ensure the stack template references the correct names
when you deploy the stack.

#### Authelia Users Database

The users database is a simple yaml file. Structure shown below.

```yaml
users:
  user@domain.com:
    displayname: "John"
    password: "$6$rounds=50000$a5B...eW1"
    email: "user@domain.com"
    groups:
      - admins
```

To generate the password hash, you can use the `mkpasswd` utility (you may need
to install it using your package manager). Use the following command:

```console
mkpasswd -m sha-512 -R 50000
```

You will be prompted to enter the password, and the hash will be output.

#### Authelia JWT Secret

Use the `openssl` command for this.

```console
openssl rand -hex 32
```

#### Authelia Session Secret

Use the `openssl` command for this.

```console
openssl rand -hex 64
```

#### Authelia Storage Encryption Key

Use the `openssl` command for this.

```console
openssl rand -hex 64
```

### 3. Deploy the stack

Once you've created all the Configs and Secrets, deploy the stack, replacing the
Config and Secret names at the top of the stack template with the names of the
ones you created.

### 4. Set up NGINX reverse proxy for Authelia

Follow the "Adding new services" section below to get an SSL certificate in
place for Authelia. **Note that you should skip step 1 this time.** The default
configuration template already has rules in place for Authelia, and it needs to
allow anonymous access since that's where you go to log in.

### 5. Set up Authelia MFA

Go to `auth.yoursite.com` and log in with the credentials you created for
yourself in the users database file. You'll be prompted to configure your MFA
token. It will tell you it's sent you an e-mail, but in fact it's just written
the e-mail contents to `/data/notification.txt` inside the container.

Use portainer as before to open a shell into the Authelia container and copy the
URL from that file. Open it in a browser to complete configuring your token.

Now you can successfully authenticate with Authelia and be allowed to access
protected services you are hosting!

The next thing you should probably do is follow the steps in "Adding new
services" to set up a secured subdomain for Portainer (and once you've done that
don't forget to remove the part of the Portainer template that exposes port
9000).

## Adding new services

When you need to add new services to the stack that have public-facing
components, you'll need to walk through a few very specific steps to get them
up and secured with SSL certificates.

### 1. Update Authelia configuration

Create a new Config for `authelia_configuration.yml` that includes rules for
securing the new service. There are commented out examples in the template.

You'll be updating the `nginx` stack in the next step, so you can wait until
then to deploy the new Config.

### 2. Stand up the HTTP server configuration

The example server configuration in the `nginx.conf` file contains two sections,
one for HTTP on port 80 and one for HTTPS on port 443. You can't start NGINX
with a configuration for HTTPS if the certificate doesn't exist yet, NGINX will
exit. So, what you need to do is create a new Config for `nginx.conf` that adds
_just_ the HTTP portion of the config, then modify the `nginx` stack to use that
new config.

### 3. Request the certificate

Once the stack is updated, use Portainer to open an interactive shell inside the
certbot container. (find the container and look for the `>_` link, select the sh
shell from the dropdown, and connect).

Use the following command in the interactive shell to request your certificate.

```console
certbot certonly --webroot -w /var/www/certbot \
    --email user@yoursite.com -d service.yoursite.com \
    --rsa-key-size 4096 --agree-tos --force-renewal
```

You should see messaging indicating success.

What's happening here is the [HTTP-01 Challenge](https://letsencrypt.org/docs/challenge-types/#http-01-challenge)

1. Let's Encrypt issues a token to certbot.
2. certbot writes that token to a folder on a volume it shares with the NGINX
   container.
3. NGINX serves the token when Let's Encrypt sends a request to the challenge
   URL.
4. This proves that you own the domain and Let's Encrypt issues the certificate.
5. certbot writes the certificate to a folder on the shared volume so NGINX can
   use it.

### 4. Stand up the HTTPS server configuration

Once you have your certificate, you can create another new Config for
`nginx.conf` that includes both the HTTP and HTTPS sections, then update the
stack with the new Config.

## Adding new users

If you need to add a new user to Authelia, you'll need to define a new Secret
for the users database. You can open a shell into the container and view the
current one at `/config/users_database.yml`. Your new user can hash their
password using the method described above and simply provide you with the hash
to include in the file.

The easiest way to help new users set up their MFA is to pull the URL from the
notification file and open it yourself, then screenshot the QR code and send it
to the new user to scan.
