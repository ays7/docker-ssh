<p align="center">
		<img src="https://raw.githubusercontent.com/serversideup/docker-ssh/main/.github/header.png" width="1200" alt="Docker Images Logo">
</p>
<p align="center">
	<a href="https://github.com/ays7/docker-ssh/blob/main/LICENSE" target="_blank"><img src="https://badgen.net/github/license/ays7/docker-ssh" alt="License"></a>
	<a href="https://github.com/serversideup/docker-ssh" target="_blank"><img src="https://img.shields.io/badge/upstream-serversideup%2Fdocker--ssh-blue" alt="Upstream"></a>
	<a href="https://github.com/sponsors/serversideup"><img src="https://badgen.net/badge/icon/Support%20Upstream?label=GitHub%20Sponsors&color=orange" alt="Support Upstream"></a>
</p>

## Introduction
This repository is a fork of [`serversideup/docker-ssh`](https://github.com/serversideup/docker-ssh), maintained at [`ays7/docker-ssh`](https://github.com/ays7/docker-ssh).

It provides a hardened SSH tunnel container based on Debian, designed exclusively for establishing secure port-forwarding tunnels into your cluster, with interactive shell sessions and command execution strictly locked down.

## Features
- 🐧 **Debian-based** - Lightweight footprint based on Debian Bookworm slim
- 🚫 **No Shell / No Command Execution** - Dedicated tunneling bastion; interactive login shells and remote commands are prohibited (`nologin` + `PermitTTY no`)
- 🎯 **Destination Restrictions** - Whitelist allowed tunnel destinations with `SSH_PERMIT_OPEN` (and listen ports with `SSH_PERMIT_LISTEN`)
- 🤝 **Key-based auth via ENV** - Grant access with the `AUTHORIZED_KEYS` environment variable
- ⛔️ **Block IPs via ENV** - Block access with the `ALLOWED_IPS` environment variable
- 🔒 **Unprivileged user** - All SSH connections are made as an unprivileged user
- 🔑 **Set your own PUID and PGID** - Have the PUID and PGID match your host user
- 🔐 **Hardened SSH** - Agent forwarding disabled by default, TTY disabled, brute force protection
- 📦 **DockerHub and GitHub Container Registry** - Choose where you'd like to pull your image from
- 🤖 **Multi-architecture** - Every image ships with x86_64 and arm64 architectures

## Usage
This is a list of the docker images this repository creates:

| Image | Image Size | Description |
| --------- | -------------------- | ----------- |
| `serversideup/docker-ssh` |[![DockerHub serversideup/docker-ssh](https://img.shields.io/docker/image-size/serversideup/docker-ssh/latest?label=latest)](https://hub.docker.com/r/serversideup/docker-ssh) | A hardened SSH tunnel server based on Debian Bookworm. |

## Usage instructions
All variables are documented here:

**🔀 Variable Name**|**📚 Description**|**#️⃣ Default Value**
:-----:|:-----:|:-----:
ALLOWED_IPS| Content of allowed IP addresses (see below)| `AllowUsers tunnel` (allow the `tunnel` user from any IP) |
AUTHORIZED_KEYS|🚨 <b>Required to be set by you.</b> Content of your authorized keys file (see below)|  |
DEBUG|Display a bunch of helpful content for debugging.|false
PGID|Group ID the SSH user should run as.|9999
PUID|User ID the SSH user should run as.|9999
SSH_GROUP|Group name used for our SSH user.|`tunnelgroup`
SSH_HOST_KEY_DIR|Location of where the SSH host keys should be stored.|`/etc/ssh/ssh_host_keys/`
SSH_PORT|Listening port for SSH server (on container only. You'll still need to publish this port).|`2222`
SSH_USER|Username for the SSH user that other users will connect into as.|`tunnel`
SSH_GATEWAYPORTS|Setting for the GatewayPorts sshd_config for reverse tunnelling|`no`
SSH_PERMIT_OPEN|Optional whitespace- or comma-separated list of `host:port` destinations allowed for port forwarding (e.g. `db:5432 redis:6379`).|unset (all destinations permitted)
SSH_PERMIT_LISTEN|Optional whitespace- or comma-separated list of listen ports/addresses for remote port forwarding (e.g. `8080`).|unset (all listen ports permitted)
SSH_ALLOW_AGENT_FORWARDING|Allow SSH agent forwarding (`yes` or `no`).|`no`


### 1. Set your `AUTHORIZED_KEYS` environment variable or provide a `/authorized_keys` file
You can provide multiple keys by loading the contents of a file into an environment variable.
```
AUTHORIZED_KEYS="$(cat .ssh/my_many_ssh_public_keys_in_one_file.txt)"
```

Or you can provide the `authorized_keys` file via a volume. Ensure the volume references matches the path of `/authorized_keys`. The image will automatically take the file from `/authorized_keys` and configure it for use with your selected user.

ℹ️ **NOTE:** If both a file and variable are provided, the image will respect the value of the **variable _over_ the file**.

### 2. Set your `ALLOWED_IPS` environment variable
Set this in the same context of [AllowUsers](https://www.ssh.com/academy/ssh/sshd_config)This example shows a few scenarios you can do:
```
ALLOWED_IPS="AllowUsers *@192.168.1.0/24 *@172.16.0.1 *@10.0.*.1"
```

### 3. Forward your external port to `2222` on the container
You can see I'm forwarding `12345` to `2222`.
```sh
docker run --rm --name=ssh --network=web -p 12345:2222 localhost/ssh
```
Because interactive shell logins and command executions are locked down, connect with `-N` (no command execution) when establishing tunnels:
```sh
ssh -N -p 12345 -L 8080:target-service:80 tunnel@myserver.test
```

# Working example with MariaDB + SSH + Docker Swarm
Here's an example of using it with MariaDB. This allows you to use Sequel Pro, TablePlus, or DBeaver to connect securely into your database server over an SSH tunnel 🥳

### Example using `ALLOWED_IPS` and `SSH_PERMIT_OPEN` variables:
```yaml
services:
  mariadb:
    image: mariadb:10.11
    networks:
      - database
    environment:
      MARIADB_ROOT_PASSWORD: "myrootpassword"

  ssh:
    image: serversideup/docker-ssh
    ports:
      - target: 2222
        published: 2222
        mode: host
    environment:
      # Set the Authorized Keys of who can connect
      AUTHORIZED_KEYS: >
        "# Start Keys
         ssh-ed25519 1234567890abcdefghijklmnoqrstuvwxyz user-a
         ssh-ed25519 abcdefghijklmnoqrstuvwxyz1234567890 user-b
         # End Keys"
      # Lock down the access to certain IP addresses
      ALLOWED_IPS: "AllowUsers tunnel@1.2.3.4"
      # Restrict forwarding destinations strictly to the database service
      SSH_PERMIT_OPEN: "mariadb:3306"
    networks:
      - database

networks:
  database:
```

### Example using `$SSH_USER_HOME/.ssh/authorized_keys` file:
```yaml
services:
  mariadb:
    image: mariadb:10.11
    networks:
      - database
    environment:
      MARIADB_ROOT_PASSWORD: "myrootpassword"

  ssh:
    image: serversideup/docker-ssh
    ports:
      - target: 2222
        published: 2222
        mode: host
    environment:
      ALLOWED_IPS: "AllowUsers tunnel@1.2.3.4"
      SSH_PERMIT_OPEN: "mariadb:3306"
    configs:
      - source: ssh_authorized_keys
        # Mount the file to "/authorized_keys". The image will handle everything else
        target: /authorized_keys
        mode: 0600
    networks:
      - database

# Define the config to be used
configs:
  ssh_authorized_keys:
    file: ./authorized_keys

networks:
  database:
```

# Tunneling Usage Examples

Interactive shells and remote command execution are disabled for security. Use the `-N` flag (or configure your GUI client for port forwarding only) to establish tunnels:

### 1. Local Port Forwarding (`-L`)
Forward a local port (e.g. `3306`) to an internal service behind the tunnel:
```sh
ssh -N -p 12345 -L 3306:mariadb:3306 tunnel@myserver.test
```

### 2. Restricting Allowed Destinations (`SSH_PERMIT_OPEN`)
You can restrict which destinations and ports clients are allowed to forward to by setting `SSH_PERMIT_OPEN` in the container environment (space- or comma-separated):
```yaml
environment:
  SSH_PERMIT_OPEN: "mariadb:3306 redis:6379 internal-api.domain:443"
```
Any attempt to forward to a non-permitted host or port will be rejected by `sshd`.

### 3. Dynamic SOCKS Proxy (`-D`)
Create a dynamic SOCKS5 proxy on local port `1080` to route traffic through the container network:
```sh
ssh -N -p 12345 -D 1080 tunnel@myserver.test
```

### 4. Remote / Reverse Port Forwarding (`-R`)
If `SSH_GATEWAYPORTS` is enabled, reverse port forwarding can expose a local service through the container:
```sh
ssh -N -p 12345 -R 8080:localhost:8080 tunnel@myserver.test
```
You can optionally restrict the remote ports clients can bind to using `SSH_PERMIT_LISTEN`:
```yaml
environment:
  SSH_GATEWAYPORTS: "yes"
  SSH_PERMIT_LISTEN: "8080"
```

## Resources
- **[GitHub Repository](https://github.com/ays7/docker-ssh)** for source code, issues, and discussions on this fork.
- **[Upstream Repository](https://github.com/serversideup/docker-ssh)** for the original project by Server Side Up.
- **[Upstream Discord](https://serversideup.net/discord)** for friendly support from the original creators and community.
- **[Get Professional Help](https://serversideup.net/professional-support)** - Get video + screen-sharing help directly from the upstream core contributors.

## Contributing
As an open-source project, we strive for transparency and collaboration in our development process. We greatly appreciate any contributions members of our community can provide. Whether you're fixing bugs, proposing features, improving documentation, or spreading awareness - your involvement strengthens the project. Please review our [code of conduct](./.github/code_of_conduct.md) to understand how we work together respectfully.

- **Bug Report**: If you're experiencing an issue while using this image, please [create an issue on GitHub](https://github.com/ays7/docker-ssh/issues/new/choose).
- **Feature Request**: Make this project better by [submitting a feature request](https://github.com/ays7/docker-ssh/discussions/).
- **Documentation**: Improve our documentation by [submitting a documentation change](./README.md).
- **Community Support**: Help others on [GitHub Discussions](https://github.com/ays7/docker-ssh/discussions) or the upstream [Discord](https://serversideup.net/discord).
- **Security Report**: Report critical security issues via [our responsible disclosure policy](https://www.notion.so/Responsible-Disclosure-Policy-421a6a3be1714d388ebbadba7eebbdc8).

Need help getting started? Join our Discord community and we'll help you out!

<a href="https://serversideup.net/discord"><img src="https://serversideup.net/wp-content/themes/serversideup/images/open-source/join-discord.svg" title="Join Discord"></a>

## Our Sponsors
All of our software is free an open to the world. None of this can be brought to you without the financial backing of our sponsors.

<p align="center"><a href="https://github.com/sponsors/serversideup"><img src="https://521public.s3.amazonaws.com/serversideup/sponsors/sponsor-box.png" alt="Sponsors"></a></p>

### Black Level Sponsors
<a href="https://sevalla.com"><img src="https://serversideup.net/wp-content/uploads/2024/10/sponsor-image.png" alt="Sevalla" width="546px"></a>

#### Bronze Sponsors
<!-- bronze -->No bronze sponsors yet. <a href="https://github.com/sponsors/serversideup">Become a sponsor →</a><!-- bronze -->

#### Individual Supporters
<!-- supporters --><p align="center"><a href="https://github.com/sponsors/serversideup"><img src="https://521public.s3.amazonaws.com/serversideup/sponsors/sponsor-empty-state.png" alt="Sponsors"></a></p><!-- supporters -->

## About Us
We're [Dan](https://twitter.com/danpastori) and [Jay](https://twitter.com/jaydrogers) - a two person team with a passion for open source products. We created [Server Side Up](https://serversideup.net) to help share what we learn.

<div align="center">

| <div align="center">Dan Pastori</div>                  | <div align="center">Jay Rogers</div>                                 |
| ----------------------------- | ------------------------------------------ |
| <div align="center"><a href="https://twitter.com/danpastori"><img src="https://serversideup.net/wp-content/uploads/2023/08/dan.jpg" title="Dan Pastori" width="150px"></a><br /><a href="https://twitter.com/danpastori"><img src="https://serversideup.net/wp-content/themes/serversideup/images/open-source/twitter.svg" title="Twitter" width="24px"></a><a href="https://github.com/danpastori"><img src="https://serversideup.net/wp-content/themes/serversideup/images/open-source/github.svg" title="GitHub" width="24px"></a></div>                        | <div align="center"><a href="https://twitter.com/jaydrogers"><img src="https://serversideup.net/wp-content/uploads/2023/08/jay.jpg" title="Jay Rogers" width="150px"></a><br /><a href="https://twitter.com/jaydrogers"><img src="https://serversideup.net/wp-content/themes/serversideup/images/open-source/twitter.svg" title="Twitter" width="24px"></a><a href="https://github.com/jaydrogers"><img src="https://serversideup.net/wp-content/themes/serversideup/images/open-source/github.svg" title="GitHub" width="24px"></a></div>                                       |

</div>

### Find us at:

* **📖 [Blog](https://serversideup.net)** - Get the latest guides and free courses on all things web/mobile development.
* **🙋 [Community](https://community.serversideup.net)** - Get friendly help from our community members.
* **🤵‍♂️ [Get Professional Help](https://serversideup.net/professional-support)** - Get video + screen-sharing support from the core contributors.
* **💻 [GitHub](https://github.com/serversideup)** - Check out our other open source projects.
* **📫 [Newsletter](https://serversideup.net/subscribe)** - Skip the algorithms and get quality content right to your inbox.
* **🐥 [Twitter](https://twitter.com/serversideup)** - You can also follow [Dan](https://twitter.com/danpastori) and [Jay](https://twitter.com/jaydrogers).
* **❤️ [Sponsor Us](https://github.com/sponsors/serversideup)** - Please consider sponsoring us so we can create more helpful resources.

## Our products
If you appreciate this project, be sure to check out our other projects.

### 📚 Books
- **[The Ultimate Guide to Building APIs & SPAs](https://serversideup.net/ultimate-guide-to-building-apis-and-spas-with-laravel-and-nuxt3/)**: Build web & mobile apps from the same codebase.
- **[Building Multi-Platform Browser Extensions](https://serversideup.net/building-multi-platform-browser-extensions/)**: Ship extensions to all browsers from the same codebase.

### 🛠️ Software-as-a-Service
- **[Bugflow](https://bugflow.io/)**: Get visual bug reports directly in GitHub, GitLab, and more.
- **[SelfHost Pro](https://selfhostpro.com/)**: Connect Stripe or Lemonsqueezy to a private docker registry for self-hosted apps.

### 🌍 Open Source
- **[AmplitudeJS](https://521dimensions.com/open-source/amplitudejs)**: Open-source HTML5 & JavaScript Web Audio Library.
- **[Spin](https://serversideup.net/open-source/spin/)**: Laravel Sail alternative for running Docker from development → production.
- **[Financial Freedom](https://github.com/serversideup/financial-freedom)**: Open source alternative to Mint, YNAB, & Monarch Money.
