# Bastion

This is a secure/locked-down bastion implemented as a Docker Container. It uses Alpine Linux as the base image and ships with support for Google Authenticator & DUO MFA support.

### MFA Setup & Usage
When using Duo as the MFA provider, this becomes even more magical because Duo supports automatic Push notifications to your mobile device.
Just approve the request on your mobile phone (e.g. with a thumb press on iOS) when prompted.

### Slack Notifications
Slack notifications for self-reporting.

* Any time a user accesses production systems, they should reply to the slack notification to justify their remote access.
* A "buddy" should approve the login by adding a reaction (e.g. ✅).
* If no one approves the login, it should trigger an *incident response* to track down the unauthorized access.
# Usage


### Running

Refer to the [Environment Variables](#environment-variables) section below to tune how the `bastion` operates.


```bash
$ docker run -p 1234:22 https://ghcr.io/jkaberg/bastion:latest
```

### Building

```bash
$ git clone https://github.com/jkaberg/bastion.git
$ cd bastion
$ make docker:build
```


### Configuration

## Recommendations

* Do not allow `root` (or `sudo`) access to this container as doing so would allow remote users to manipulate audit-logs in `/var/log/sudo-io`
* Use the bastion as a "jump host" for accessing other internal systems rather than installing a lot of unnecessary stuff, which increases the overall attack surface.
* Sync the contents of `/var/log/sudo-io` to a remote, offsite location. If using S3, we recommend enabling bucket-versioning.
* Bind-mount `/etc/passwd`, `/etc/shadow` and `/etc/group` into the container as *read-only*
* Bind-mount `/home` into container; the bastion does not manage authorized keys

#### Environment Variables

The following tables lists the most relevant environment variables of the `bastion` image and their default values.

##### Duo Settings

Duo is a enterprise MFA provider that is very affordable. Details here: https://duo.com/pricing


| ENV               |      Description                                    |  Default |
|-------------------|:----------------------------------------------------|:--------:|
| `MFA_PROVIDER`    |  Enable the Duo MFA provider                        | duo      |
| `DUO_IKEY`        |  Duo Integration Key                                |          |
| `DUO_SKEY`        |  Duo Secret Key                                     |          |
| `DUO_HOST`        |  Duo Host Endpoint                                  |          |
| `DUO_FAILMODE`    |  How to fail if Duo cannot be reached               | secure   |
| `DUO_AUTOPUSH`    |  Automatically send a push notification             | yes      |
| `DUO_PROMPTS`     |  How many times to prompt for MFA                   | 1        |


##### Google Authenticator Settings

Google Authenticator is a free & open source MFA solution. It's less secure than Duo because tokens are stored on the server under each user account.


| ENV               |      Description                                    |  Default              |
|-------------------|:----------------------------------------------------|:---------------------:|
| `MFA_PROVIDER`    |  Enable the Google Authenticator provider           | google-authenticator  |


##### Enforcer Settings

The enforcer ensures certain conditions are satisfied. Currently, these options are supported.

| ENV                           |  Description                                                 |  Default |
|-------------------------------|:-------------------------------------------------------------|:--------:|
| `ENFORCER_ENABLED`            |  Enable general enforcement                                  | `true`   |
| `ENFORCER_CLEAN_HOME_ENABLED` |  Erase dot files in home directory before starting session   | `true`   |

##### Slack Notifications

The enforcer is able to send notifications to a slack channel anytime there is an SSH login.

| ENV                        |      Description                                    |  Default  |
|----------------------------|:----------------------------------------------------|:---------:|
| `SLACK_ENABLED`            | Enabled Slack integration                           | `false`   |
| `SLACK_HOOK`               | Slack integration method (e.g. `pam`, `sshrc`)      | `sshrc`   |
| `SLACK_WEBHOOK_URL`        | Webhook URL                                         |           |
| `SLACK_USERNAME`           | Slack handle of bot (defaults to short-dns name)    |           |
| `SLACK_TIMEOUT`            | Request timeout                                     | `2`       |
| `SLACK_FATAL_ERRORS`       | Deny logins if slack notification fails             | `true`    |


##### SSH Auditor

The SSH auditor uses [`sudosh`](https://github.com/cloudposse/sudosh/) to record entire SSH sessions (`stdin`, `stdout`, and `stderr`).


| ENV                   |      Description                                    |  Default     |
|-----------------------|:----------------------------------------------------|:------------:|
| `SSH_AUDIT_ENABLED`   |  Enable the SSH Audit facility                      | `true`       |

This will require that users login with the `/usr/bin/sudosh` shell.

Update user's default shell by running the command: `usermod -s /usr/bin/sudosh $username`. By default, `root` will automatically be updated to use `sudosh`.

Use the `sudoreplay` command to audit/replay sessions.


#### User Accounts & SSH Keys

The `bastion` does not attempt to manage user accounts.

### Extending

The `bastion` was written to be easily extensible.

You can extend the enforcement policies by adding shell scripts to `etc/enforce.d`. Any scripts that are `+x` (e.g. `chmod 755`) will be executed at runtime.


## Quick Start


Here's how you can quickly demo the `bastion`. We assume you have `~/.ssh/authorized_keys` properly configured and your SSH key (e.g. `~/.ssh/id_rsa`) added to your SSH agent.


```bash
$ docker run -it -p 1234:22 \
     -e MFA_PROVIDER=google-authenticator \
     -v ~/.ssh/authorized_keys:/root/.ssh/authorized_keys \
     https://ghcr.io/jkaberg/bastion
```

Now, in another terminal you should be able to run:
```bash
$ ssh root@localhost -p 1234
```

The first time you connect, you'll be asked to setup your MFA device. Subsequently, each time you connect, you'll be prompted to enter your MFA token.