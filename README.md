# Docker DNS-gen

dns-gen sets up a container running Dnsmasq and [docker-gen].
docker-gen generates a configuration for Dnsmasq and reloads it when containers are
started and stopped.

By default it will provide thoses hosts: `containername.docker` and `servicename.projectfolder.docker`
pointing to the corresponding container.

## Start the container

It is possible to use either using the `dns.tld` label or the `DOMAIN_TLD` environment variable.
If none of these is set it will default to `docker`.
Only the first found will be used, the label will be tried first then the environment.

    $ docker run -d --name dns-gen \
      --publish 54:53/udp \
      --volume /var/run/docker.sock:/var/run/docker.sock:ro \
      --label dns.tld=my.tld
      --environment DOMAIN_TLD=my.other.tld
      zetaron/docker-dns-gen

## Ways to configure domain names on your containers

There are two ways to assign a domain to a container.
By default all containers are built using the TLD configured on the dns-gen container.

### 1. jwilder/nginx-proxy compatible way

This one will ignore the TLD configured on the dns-gen container.

    $ docker run \
      --environment VIRTUAL_HOST=test.docker
      dockercloud/hello-world

### 2. ahmetalpbalkan/wagl compatible way

The following statement respects the TLD configured on the dns-gen container.

    $ docker run \
      --label dns.service=test \
      --label dns.domain=project \
      dockercloud/hello-world

The domain for the container started above will be `test.project.TLD` where TLD is the TLD configured on your dns-gen container.


The next example also respects the TLD configured on the dns-gen container, but adds it's own.

    $ docker run \
      --label dns.service=test \
      --label dns.domain=project \
      --label dns.tld=awesome \
      dockercloud/hello-world

This leads to the follwoing two domains pointing to the conatiner:
1. test.project.TLD (TLD you set on the dns-gen conatiner)
2. test.project.awesome (using the TLD you set on this container)

## Start the container automatically after reboot

You can tell docker (version >= 1.2) to automatically start the DNS container
after booting, by passing the option `--restart always` to your `run` command.

    $ docker run -d --name dns-gen \
      --restart always \
      --publish 54:53/udp \
      --volume /var/run/docker.sock:/var/run/docker.sock:ro \
      zetaron/docker-dns-gen

**beware**! When your host will restart, it may change the IP address of
the `docker0` interface.
This small change will prevent docker to start your dns-gen container.  Indeed,
remember our container is configured to forward port 53 to the previous
`docker0` interface which may not exist after reboot.  Your container just will
not start, you will have to re-create it. To solve this drawback, force docker
to always use the same IP range by editing the default configuration of the docker
daemon (sometimes located in `/etc/default/docker` but may change regarding
your distribution). You have to restart the docker service to take the changes
into account. Sometimes the interface is not updated, you will have to restart
your host.

    # For systemd users (Fedora and recent Ubuntu versions) :
    $ vim /lib/systemd/system/docker.service
    # append the --bip="172.17.42.1/24" option to the ExecStart line
    # then
    $ sudo systemctl daemon-reload

    # For other users
    $ vim /etc/default/docker

    DOCKER_OPTS="--bip=172.17.42.1/24"

    # In any cases
    $ sudo service docker restart

**One more thing** When you start your host, the docker service is not fully
loaded.
Until this daemon is loaded, the dns container will not be automatically started
and you will notice bad performance when your host will try to resolve DNS.
The service is not fully loaded, because it uses a feature of systemd called
[socket activation]: The first access to the docker socket will trigger the
start of the true service.
To skip this feature, you simply have to activate the docker service.

    $ sudo update-rc.d docker enable

Et voila, now, docker will really start with your host, it will always
use the same range of IP addresses and will always start/restart the container
dns-gen.

### Advanced usage

Instead of using the `docker0` interface and modifying `/etc/resolv.conf`,
an other solution is to install localy a dnsmasq server (some distribs like
ubuntu or debian are now using it by default) and forward requests to the
dns-gen container and configure containers to use it.

*step 1* Configure the local dnsmasq to forward request to `127.0.0.1:54`
And listen to interfaces `lo` and `docker0`.

    $ sudo vim /etc/NetworkManager/dnsmasq.d/01_docker`
    bind-interfaces
    interface=lo
    interface=docker0

    server=/docker/127.0.0.1#54

    $ sudo systemctl status NetworkManager

*step 2* Run dns-gen and bind port `53` to the `54`'s host

    $ docker run --daemon --name dns-gen \
      --restart always \
      --publish 54:53/udp \
      --volume /var/run/docker.sock:/var/run/docker.sock:ro \
      zetaron/docker-dns-gen -R

> the option `-R` just tell dns-gen to not fallback to the default resolver
> which avoid an infinity loop of resolution


*step 3* Configure docker to use the `docker0` as DNS server

    # For systemd users (Fedora and recent Ubuntu versions) :
    
    $ vim /etc/systemd/system/docker.service
    
    # append
    [Service]
    ExecStart=
    ExecStart=/usr/bin/docker daemon -H fd:// --bip=172.17.42.1/24 --dns=172.17.42.1
    
    # then
    $ sudo systemctl daemon-reload

    # For other users
    $ vim /etc/default/docker
    DOCKER_OPTS="--dns=172.17.42.1 --bip=172.17.42.1/24"

    # In any cases
    $ sudo service docker restart

Thank to this configuration the resolution workflow is now:

 * the host want to resolve `google.com`: `host` -> `dnsmasq` -> `external dns`
 * the host want to resolve `foo.docker`: `host` -> `dnsmasq` -> `127.0.0.1:54` -> `dns-gen`
 * a container want to resolve `google.com`: `container` -> `172.17.42.1` -> `dnsmasq` -> `external dns`
 * a container want to resolve `foo.docker`: `container` -> `172.17.42.1` -> `dnsmasq` -> `127.0.0.1:54` -> `dns-gen`


  [docker-gen]: https://github.com/jwilder/docker-gen
  [socket activation]: http://0pointer.de/blog/projects/socket-activation.html
