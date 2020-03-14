---
title: "Install Nextcloud on RHEL/CentOS 8 using Podman Pods"
date: 2020-02-26T15:00:00
draft: true
toc: true
images:
tags:
  - centos
  - rhel
  - podman
  - docker
  - nextcloud
  - how-to
  - guide
---

## Summary

Since Red Hat dropped support for Docker entirely with the release of Red Hat Enterprise Linux 8, and therefore also CentOS 8, there's a big move over to it's replacement container CLI: [Podman](https://podman.io).

However, while the CLI for simply running containers is the same and acts as a full drop in replacement for Docker, there are some things still missing that people got used to. Probably the most dominant feature that's missing is [docker-compose](https://docs.docker.com/compose/) and therefore the ability to easily deploy and manage whole applications.

While pods aren't intended to bring the same functionality as docker-compose, it's possible to easily use pods to deploy self-contained applications on a host running Podman. This guide is supposed to demonstrate how to do this with nextcloud and the accompanying MariaDB database, while persistently keeping all important files and showing how to update the containers.

This guide is using CentOS 8.1 and Podman v1.6.4.

## tl;dr

1. Create pod

    ```bash
    podman pod create --name nextcloud -p 8080:80
    ```

2. Create directory for persistent data

    ```bash
    mkdir -p /srv/nextcloudapp
    mkdir -p /srv/nextclouddb
    ```

3. Set correct SELinux context

    ```bash
    semanage fcontext -a -t container_file_t "/srv/nextcloudapp(/.*)?"
    semanage fcontext -a -t container_file_t "/srv/nextclouddb(/.*)?"
    restorecon -rv /srv/nextcloud*
    ```

4. Run containers in the pod with persistent data

    ```bash
    podman container run -d --name nextcloudapp --pod nextcloud -v /srv/nextcloudapp:/var/www/html nextcloud:stable

    podman container run -d --name nextclouddb --pod nextcloud -v /srv/nextclouddb:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=changeme -e MYSQL_PASSWORD=changeme -e MYSQL_DATABASE=nextcloud -e MYSQL_USER=nextcloud mariadb:latest
    ```

5. Export pod configuration for later use

    ```bash
    podman generate kube nextcloud > nextcloud-pod.yml
    ```

## Installing Podman

Podman is available via the default repositories on RHEL/CentOS 8.

```bash
dnf install -y podman
```

## What is a Pod?

Pods come from the Kubernetes world and are a group of one or more containers with shared storage and network. The full explanation would exceed this guide and is available in the [Kubernetes Documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod/). In my opinion, the key difference when going from docker-compose to pods is how networking is handled:

- All containers in a Pod share one network
- Containers can communicate with each other over localhost
- Ports are exposed by the Pod, not by the containers

As a result of all this, each (internal) port in a Pod can only be used once by any given container.

## Creating the Nextcloud Pod

When creating the Pod, it's important to expose the needed ports to the host. The Nextcloud container listens to port 80, which we'll expose to the hosts port 8080. The syntax is the same as docker.

For ease of use, we'll also give the Pod a name.

```bash
podman pod create --name nextcloud -p 8080:80
```

It should now already show us the Pod it created with `podman pod ls`:

```bash
$ podman pod ls
POD ID         NAME        STATUS    CREATED        # OF CONTAINERS   INFRA ID
ca6dfdb74a3b   nextcloud   Created   1 minute ago   1                 8c0e1c8427de
```

As we can see, it contains one container by default. This is the [pause container](https://stackoverflow.com/a/48654926), which holds the network namespace for the Pod. We can list all running containers and see it's respective Pod with `podman ps -a --pod`:

```bash
$ podman ps -a --pod
CONTAINER ID  IMAGE                 COMMAND  CREATED       STATUS            PORTS                 NAMES               POD
8c0e1c8427de  k8s.gcr.io/pause:3.1           1 minute ago  Up 3 minutes ago  0.0.0.0:8080->80/tcp  ca6dfdb74a3b-infra  ca6dfdb74a3b
```

## Creating persistent storage mounts

Since a single container isn't meant to live forever, it's important to export any data that is meant to be kept. Therefore, we'll create two storage mounts: One for the database and one for Nextcloud's data itself.

```bash
mkdir -p /srv/nextcloudapp
mkdir -p /srv/nextclouddb
```

As SELinux would prevent the containers from accessing these directories, we need to set the correct SELinux context

```bash
semanage fcontext -a -t container_file_t "/srv/nextcloudapp(/.*)?"
semanage fcontext -a -t container_file_t "/srv/nextclouddb(/.*)?"
restorecon -rv /srv/nextcloud*
```

If you aren't running the podman containers as root (which is one of the key advantages Podman has from Docker), make sure to `chown` these directories to the correct user as well.

## Creating the Nextcloud and Database Container

To make sure that both containers will be launched inside the Pod, we'll make use of the `--pod` argument and pass it the Pods name.

Now it is also important to mount the persistent storage folders we created in the previous step to the correct paths inside the Container. For this, we make use of the docker familiar `-v` flag.

At this stage, we also set the credentials for the Database container. Be responsible and use a more secure password than this guide does.

If you're not running these containers as root, make sure to `chown $USER:$USER /srv/nextcloud{app,db}` so the containers can write to it.

```bash
podman container run -d --name nextcloudapp --pod nextcloud -v /srv/nextcloudapp:/var/www/html nextcloud:stable

podman container run -d --name nextclouddb --pod nextcloud -v /srv/nextclouddb:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=changeme -e MYSQL_PASSWORD=changeme -e MYSQL_DATABASE=nextcloud -e MYSQL_USER=nextcloud mariadb:latest
```

At this point we can already access the Nextcloud web interface and go through the setup wizard. For this, simply access http://yourcontainerhostip:8080. Keep in mind that firewalld is up and running by default, so you may want to run `firewall-cmd --zone=public --add-port=8080/tcp` to be able to access the site.

Now you can create your admin user and set up the database:

![nextcloud-init-setup](/images/nextcloud-initsetup.png)

Make sure to set the username, password and database name to the respective values set above as environment variables for the MariaDB container.

I also found that it's necessary to use `127.0.0.1` as database server. `localhost` did not work for me.

## Nextcloud Webcron Job

Nextcloud frequently requires background jobs to be run. The recommended time is every 5 minutes. I found the easiest way for this is to use Webcron and call the nextcloud webpage every 5 minutes via the hosts cron daemon:

1. In the Nextcloud Webinterface, set the Background jobs to Webcron

    ![nextcloud-webcron](/images/nextcloud-webcron.png)

2. Add the following cronjob to your host via `crontab -e`

    ```bash
    */5 * * * * curl localhost:8080/cron.php
    ```

## Automatically start the Pod on boot

Podman, unlike Docker, doesn't run a daemon that takes care of automatically starting on boot or restarting Pods/Containers in case of a failure. With Podman, systemd is meant to take care of this job. Therefore it's necessary to create unit files for the Containers and Pods.

### Add rootless pods to systemd

If you created the Containers and Pods as a typical non-root user, you can't add the Pods to the normal "global" system daemon. This is because that daemon wouldn't run the Pod as your user and therefore couldn't find them. In this case it's necessary to add it to the users individual daemon

1. Create the directory for user specific unit files

    ```bash
    mkdir -p ~/.config/systemd/user/
    ```

2. Generate the systemd unit files

    ```bash
    podman generate systemd -f --name nextcloud
    ```

3. Replace `WantedBy=multi-user.target` with `WantedBy=default.target` in the `pod-nextcloud.service` file

    ```bash
    sed -i 's/WantedBy=multi-user.target/WantedBy=default.target/' pod-nextcloud.service
    ```

4. Copy the files to `~/.config/systemd/user/`

    ```bash
    cp {container-nextcloudapp,container-nextclouddb,pod-nextcloud}.service ~/.config/systemd/user/
    ```

5. Reload the daemon and enable the service

    ```bash
    systemctl --user daemon-reload
    systemctl --user enable pod-nextcloud.service
    ```

6. Enable user lingering. Otherwise your service would only start upon login of your user and not with system boot

    ```bash
    loginctl enable-linger $USER
    ```

### Add root pods to systemd

When the containers were created as the root user, it works like any other unit files as well.

1. Generate the systemd unit files

    ```bash
    podman generate systemd -f --name nextcloud
    ```

2. Copy the files to `/etc/systemd/system/`

    ```bash
    cp {container-nextcloudapp,container-nextclouddb,pod-nextcloud}.service /etc/systemd/system
    ```

3. Reload the daemon and enable the service

    ```bash
    systemctl daemon-reload
    systemctl enable pod-nextcloud.service
    ```

## Updating

## Collaboration