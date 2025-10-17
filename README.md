# ssh2lxd – SSH server for LXD/Incus containers

_Note: This is a fork of [artefactcorp/ssh2lxd](https://github.com/artefactcorp/ssh2lxd), which hasn’t been updated for several years and no longer builds with modern Go. Since no one appears to be maintaining that project or accepting pull requests, I’ve applied my changes here._

**ssh2lxd** is an SSH server that allows direct connections into LXD containers.
It uses the LXD API to establish a connection with a container and create a session.

My use case is allowing configuration management tools (e.g. Ansible, Pyinfra, Chef) to deploy directly into Incus containers without having to run an SSH daemon in each container.

## Features

- Authentication using existing host OS SSH keys via `authorized_keys`
- SSH Agent forwarding into a container session
- Full support for PTY (terminal) mode and remote command execution
- Support for SCP and SFTP*
- Doesnt require root to run, if you allow access to the LXD socket

> *SFTP support relies on `sftp-server` binary being installed in the container (see below)

## Changes made in this fork

- Allow specifying an SSH host key file instead of randomly generating an RSA key at every startup.
- Use a newer `github.com/gliderlabs/ssh` via `go.mod` instead of the bundled static version.
  This was also necessary to support non-deprecated host key types like Ed25519.

## Install / Building from source

```bash
go build ./cmd/ssh2lxd/
```

Generate a host key with something like:
```bash
mkdir /home/ssh2lxd/.ssh2lxd/ # or wherever you want it stored.
ssh-keygen -t ed25519 -f /home/ssh2lxd/.ssh2lxd/hostkey_ed25519 -N ''
```

#### Setting up ssh2lxd as a service

Heres an example systemd unit file that you could add as eg. `/etc/systemd/system/ssh2lxd.service`:

```ini
[Unit]
Description=SSH to LXD bridge (ssh2lxd)
After=network.target

[Service]
ExecStart=/home/pyinfra/ssh2lxd -s /var/lib/incus/unix.socket -d -g ssh2lxd --hostkey /home/ssh2lxd/.ssh2lxd/ed25519_host_key
WorkingDirectory=/home/ssh2lxd
Restart=on-failure
User=ssh2lxd

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Change `User` and `WorkingDirectory` to match your setup.

Customize `ExecStart`:
- `-s` to point to your lxd/incus socket
- make `--hostkey` point to your to host-key file.
- remove `-d` (debug) if you dont want it in production

```
systemctl enable ssh2lxd.service
systemctl start ssh2lxd.service
```

#### Checking logs

```bash
journalctl -f -u ssh2lxd.service
```

## Basic Connection

To establish an SSH connection to a container running on LXD host, run:

```bash
ssh -p 2222 [host-user+]container-name[+container-user]@lxd-host
```

and substitute the following

- `host-user` – active user on LXD host such as `root`
- `container-name` – running container on LXD host
- `container-user` – active user in LXD container (_optional, defaults to_ `root`)
- `lxd-host` – LXD host hostname or IP

### Examples

To connect to a container `ubuntu` running on LXD host with IP `1.2.3.4` as `root` user and authenticate
as `root` on LXD host, run:

```bash
ssh -p 2222 ubuntu@1.2.3.4
```

To connect to a container `ubuntu` running on LXD host with IP `1.2.3.4` as `root` user and authenticate
as `admin` on LXD host, run:

```bash
ssh -p 2222 admin+ubuntu@1.2.3.4
```

To connect to a container `ubuntu` running on LXD host with IP `1.2.3.4` as `ubuntu` user and authenticate
as `root` on LXD host, run:

```bash
ssh -p 2222 root+ubuntu+ubuntu@1.2.3.4
```

## Advanced Connection

### SSH Agent forwarding

`ssh2lxd` supports SSH Agent forwarding. To make it work in a container, it will automatically add a
proxy socket device to LXD container and remove it once SSH connection is closed.

_Note of warning: SSH Agent Forwarding inherently allows the remote container, to use your agent to log in to other hosts, so you need to trust that the host or container has not been compromised. I generally advise against using SSH Agent Forwarding._


To enable SSH agent on your local system, run:

```
eval `ssh-agent`
```

To enable SSH Agent forwarding when connecting to a container add `-A` to your `ssh` command

```
ssh -A -p 2222 ubuntu@1.2.3.4
```

### Using LXD host as SSH Proxy / Bastion

You can access an LXD container by using LXD host's SSH server as a Proxy / Bastion.
The easiest way is to add additional configuration to your `~/.ssh/config`

```
Host lxd1
  Hostname localhost
  Port 2222
  ProxyJump lxd-host

Host lxd-host
  Hostname 1.2.3.4
  User root
```

Now to connect to `ubuntu` container as `root`, run:

```
ssh ubuntu@lxd1
```

> Using this method has additional security benefits and port 2222 is not exposed to the public

### SFTP Connection

In order to enable full SFTP support on an LXD container it needs `sftp-server` binary installed. And it doesn't require
`sshd` service to run in a container.

#### Ubuntu / Debian containers

```
apt-get update
apt-get install openssh-sftp-server
```

#### CentOS / Fedora containers

```
yum install openssh-server
```

#### Alpine Linux containers

```
apk update
apk add openssh-sftp-server
```

### Ansible

Running Ansible commands and playbooks directly on LXD containers is fully support with or without `sftp-server` binary
in a container. Ansible falls back to SCP mode when SFTP is not available.

#### Examples

```
ansible.cfg:

[defaults]
host_key_checking = False
remote_tmp = /tmp/.ansible-${USER}
```

```
inventory:

# Direct connection to port 2222
[lxd1]
container-a ansible_user=root+c1 ansible_host=1.2.3.4 ansible_port=2222
container-b ansible_user=root+u1+ubuntu ansible_host=1.2.3.4 ansible_port=2222 become=yes

# Connection using ProxyJump configured in ssh config
[lxd2]
container-c ansible_user=root+c1 ansible_host=lxd1
container-d ansible_user=root+u1+ubuntu ansible_host=lxd1 become=yes
```

```
playbook.yml:

---
- hosts: lxd1,lxd2
  become: no
  become_method: sudo

  tasks:
    - command: env
    - command: ip addr
```


## Configuration Options

By default `ssh2lxd` will listen on port `2222` and allow authentication for `root` and users who belong to the groups
`wheel,lxd`.

To add a user to one of those groups run as root `usermod -aG lxd your-host-user`

```
-d, --debug                enable debug log
-g, --groups string        list of groups members of which allowed to connect (default "wheel,lxd")
    --healthcheck string   enable LXD health check every X minutes, e.g. "5m"
-h, --help                 print help
    --hostkey string       SSH host key file
-l, --listen string        listen on :2222 or 127.0.0.1:2222 (default ":2222")
    --noauth               disable SSH authentication completely
-s, --socket string        LXD socket or use LXD_SOCKET (default "/var/snap/lxd/common/lxd/unix.socket")
-v, --version              print version
```

For example, to enable debug log and listen on localhost add `-d -l 127.0.0.1:2222` to the command line.

For Debian/Ubuntu, groups set to -g "adm,lxd" is more fitting.


### Firewall

If you have firewall enabled on your LXD host, you may need to allow connections to port `2222`

