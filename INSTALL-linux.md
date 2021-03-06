# Installing the CircleCI Runner on Linux

## Create the runner configuration
The recommended runner configuration for Linux is:
```yaml
api:
  auth_token: AUTH_TOKEN
runner:
  name: RUNNER_NAME
  command_prefix: ["/opt/circleci/launch-task"]
  working_directory: /opt/circleci/workdir/%s
  cleanup_working_directory: true
```

## Install the runner configuration
Once created save the configuration file to `/opt/circleci/launch-agent-config.yaml` owned by `root` with permissions `600`.

```bash
sudo chown root: /opt/circleci/launch-agent-config.yaml
sudo chmod 600 /opt/circleci/launch-agent-config.yaml
```

## Create the circleci user & working directory
These will be used when executing the `build-agent`.

```bash
id -u circleci &>/dev/null || adduser --uid 1500 --disabled-password --gecos GECOS circleci

mkdir -p /opt/circleci/workdir
chown -R circleci /opt/circleci/workdir
```

## Install the Launch Script

This wrapper script will be used by Launch Agent to execute the Task Agent, while ensuring appropriate sandboxing and a clean shutdown.

Create `/opt/circleci/launch-task` owned by `root` with permissions `755`

```bash
#!/bin/bash

set -euo pipefail

## This script launches the build-agent using systemd-run in order to create a
## cgroup which will capture all child processes so they're cleaned up correctly
## on exit.

# The user to run the build-agent as - must be numeric
USER_ID=$(id -u circleci)

# Give the transient systemd unit an inteligible name
unit="circleci-$CIRCLECI_LAUNCH_ID"

# When this process exits, tell the systemd unit to shut down
abort() {
  if systemctl is-active --quiet "$unit"; then
    systemctl stop "$unit"
  fi
}
trap abort EXIT

systemd-run \
    --pipe --collect --quiet --wait \
    --uid "$USER_ID" --unit "$unit" -- "$@"
```

## Enable the systemd unit

Create `/opt/circleci/circleci.service` owned by `root` with permissions `755`.

You must ensure that `TimeoutStopSec` is greater than the total amount of time the `stop-agent` script will take.

If you want to configure the CircleCI runner installation to start on boot, it is important to note that the Launch Agent will attempt to consume and start jobs as soon as it starts, so it should be configured appropriately before starting. The Launch Agent may be configured as a service and be managed by systemd with the following scripts:

```
[Unit]
Description=CircleCI Runner
After=network.target
[Service]
ExecStart=/opt/circleci/circleci-launch-agent --config /opt/circleci/launch-agent-config.yaml
Restart=always
User=root
NotifyAccess=exec
TimeoutStopSec=600
[Install]
WantedBy = multi-user.target
```

You can now enable the service:
```bash
prefix=/opt/circleci
systemctl enable $prefix/circleci.service
```

## Start the service

When the CircleCI runner Service starts, it will immediately attempt to start running jobs, so it should be fully configured before the first start of the service.

```bash
systemctl start circleci.service
```

## Verify the service is running

The system reports a very basic health status through the `Status` field in `systemctl`.
This will report **Healthy** or **Unhealthy** based on connectivity to the CircleCI APIs.

You can see the status of the agent by running:

```bash
systemctl status circleci.service --no-pager
```

Which should produce output similar to:

```
● circleci.service - CircleCI Runner
   Loaded: loaded (/opt/circleci/circleci.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2020-05-29 14:33:31 UTC; 18min ago
 Main PID: 5592 (circleci-launch)
   Status: "Healthy"
    Tasks: 8 (limit: 2287)
   CGroup: /system.slice/circleci.service
           └─5592 /opt/circleci/circleci-launch-agent --config /opt/circleci/launch-agent-config.yaml
```

You can also see the logs for the system by running:

```bash
journalctl -u circleci
```
