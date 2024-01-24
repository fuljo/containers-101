# Containers 101

This is a quick guide to deploy a Splunk container on your local machine.

This is meant to be an office demo, for a complete guide see the [docker-splunk](https://splunk.github.io/docker-splunk/) project.

## Prequisites

First install Podman following [the official docs](https://podman.io/docs/installation).

Then clone this repo on your local machine:

```sh
git clone https://github.com/fuljo/containers-101.git
cd containers-101
```

## Basic deployment

### Getting the image

Now you need to find the image you want to deploy.\
You can visit a public container registry like [Docker Hub](https://hub.docker.com) and search for "Splunk".\
Various repositories will pop up, usually containing a README with a quick-start guide.\
For this demo, we will use the official image `splunk/splunk` from Docker Hub.

Once you have located the _name_ of the image, you must also select a specific _tag_, which usually specifies the version of the application.\
In development you can use the `latest` tag, but in production it is recommended to pick a specific version.
In this demo, we will use version `9.0`.

You can now pull the image on your local machine.
We will use a fully-qualified image reference, including the hostname of the registry.

```sh
podman pull docker.io/splunk/splunk:9.0
```

You can show all the images on your local system with:

```sh
podman image ls
```

### Running the container

Now you can start a minimal, standalone instance of Splunk with the following command:

```sh
podman run \
    -d \
    --name splunk \
    -p 8000:8000 \
    -e "SPLUNK_START_ARGS=--accept-license" \
    -e "SPLUNK_PASSWORD=<password>" \
    splunk/splunk:9.0
```

What's going on:

- the `-d` flag specifies that the container shall be run in **detached mode**, i.e. in the background without attaching to your terminal.
- the `--name` explicitly sets a name for the container, otherwise a random one would be generated
- the `-p` option specifies that **port** `8000` inside the container (right) shall be exposed as port `8000` (left) on the host.
- the `-e` options are setting **environment** variables inside the container

As output, Podman prints out the ID of the created container.

After the container starts up, you can access Splunk Web at http://localhost:8000 with admin:\<password\>.

### Managing the container's lifecycle

You can verify that the container is running by listing all the running containers:

```sh
podman ps
```

To stop it, simply run:

```sh
podman stop splunk
```

The container is now stopped, but it hasn't been deleted from the system.
You can verify it by listing all containers:

```sh
podman ps -a
```

You can restart it by running:

```sh
podman start splunk
```

Finally you can delete the container, which will cause you to lose its state.

```sh
podman stop splunk
podman rm splunk
```

## Advanced usage

### Mounting a configuration file

Let's say that you want to provide Splunk with a custom `default.yml` configuration file from the current directory.
You can use a **bind mount** to achieve such goal.

In general, you can mount a `<host-path>` to a `<container-path>` by providing the following parameter to the `podman run` command.

```sh
podman run \
    -v "<host-path>:<container-path>"
    ...
```

In our case:

```sh
    -v "$(pwd)/default.yml:/tmp/defaults/default.yml"
```

### Persisting data

The container that we have created is storing its data (indexes, search artifacts, ...) inside its local filesystem.
Therefore, when we delete the container such data will be lost.

It's best practice to deploy a container as a **stateless** entity, so that all its state (configuration and data) is stored externally.

Inside the container, the data is stored in the `/var` directory.
So a first approach would be to create a local directory named `data` and mount it in the container, like this:

```sh
podman run \
    -v "$(pwd)/data:/opt/splunk/var"
    ...
```

A more flexible approach is to use **named volumes**, which are managed by Podman.
By default, these are created on the local filesystem, but with additional drivers one can also mount NFS shares or virtual drives.

In our case, we create a volume named `splunk-data` on the local filesystem.

```sh
podman volume create splunk-data
```

Then mount it with the same option as bind mounts:

```sh
podman run \
    -v "splunk-data:/opt/splunk/var"
    ...
```

You can see the location at which the data is actually stored by running:

```sh
podman volume inspect splunk-data
```

### Using Podman Secrets

Podman has the ability to store secrets and expose them inside containers.
This is useful to provide passwords as well as cryptographic keys and certificates.

For an introduction to secrets, read [this article](https://www.redhat.com/sysadmin/new-podman-secrets-command).

For our Splunk container, create a secret for the administrator's password.

```
podman secret create splunk-password -
```

Then you can expose the secret inside the container as follows

```sh
podman create \
	--secret splunk-password, \
         type=env, \
         target=SPLUNK_PASSWORD \
	...
```

Of course, you shall remove the redundant option `-e "SPLUNK_PASSWORD=<password>"`.

### Running a container as a systemd service

Podman 4.4 introduced a convenient way, called Quadlet, to completely manage a container as a systemd service.
You can read an [introductory article](https://www.redhat.com/sysadmin/quadlet-podman) or the full [manual page](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html).

Make sure you have completely stopped and removed the splunk container from the previous steps.
Now let's create a unit file named `splunk.container` in either:

- `$HOME/.config/containers/systemd/` if you want to run the container as a regular user
- `/etc/containers/systemd/` if you want to run it as root

The unit file specifies all the configuration options which were passed to the `podman run` command.

```systemd
[Unit]
Description=Splunk Platform
Wants=network-online.target
After=network-online.target

[Service]
Restart=on-failure

[Install]
WantedBy=default.target

[Container]
ContainerName=splunk
Image=splunk/splunk:9.0
Environment=”SPLUNK_START_ARGS=--accept-license”
Secret=splunk-password,type=env,target=SPLUNK_PASSWORD
PublishPort=8000:8000
Volume=<your-splunk-dir>/default.yml:/tmp/defaults/default.yml
Volume=splunk-data:/opt/splunk/var
```

Remember to substitute `<your-splunk-dir>` with the directory where your `default.yml` resides.

Now if you reboot the machine, the container will start automatically.

## Troubleshooting

### Inspecting container logs

By default, Podman captures the stdout and stderr of the container.
These can be inspected by using:

```sh
podman logs splunk      # cat mode
podman logs -f splunk   # tail -f mode
```

### Launching a shell

You can execute a single command inside a running container with the following syntax:

```sh
podman exec <container-name> <command>
```

For example, to launch an interactive shell inside our container:

```sh
podman exec -it splunk bash
```
