# play-with-docker - JetLabs Fork

Play With Docker gives you the experience of having a free Alpine Linux Virtual Machine in the cloud
where you can build and run Docker containers and even create clusters with Docker features like Swarm Mode.

Under the hood DIND or Docker-in-Docker is used to give the effect of multiple VMs/PCs.

A live version is available at: http://play-with-docker.com/

## Requirements

Docker 1.13+ is required. 

The docker daemon needs to run in swarm mode because PWD uses overlay attachable networks. For that
just run  `docker swarm init` in the destination daemon.

It's also necessary to manually load the IPVS kernel module because as swarms are created in `dind`, 
the daemon won't load it automatically. Run the following command for that purpose: `sudo modprobe xt_ipvs`


## Pre-built Containers

This fork is a snapshot of the original play-with-docker repository of a fully working revision. To get this easily up and running we provide pre-built containers on Jetlabs' docker hub. To run, use the following docker compose definition:

```
version: '3.2'
services:
    haproxy:
        container_name: haproxy
        image: haproxy
        ports:
            - "80:8080"
        volumes:
            - ./haproxy:/usr/local/etc/haproxy
            
    pwd:
        # pwd daemon container always needs to be named this way
        container_name: pwd
        # use the latest pre-built image
        image: jetlabs/pwd
        volumes:
            # since this app creates networks and launches containers, we need to talk to docker daemon
            - /var/run/docker.sock:/var/run/docker.sock
            # mount the box mounted shared folder to the container
            - sessions:/pwd
    l2:
        container_name: l2
        # use the latest pre-built image
        image: jetlabs/l2
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - networks:/pwd
        ports:
            - "8022:22"
            - "8053:53"
            - "443:443"
volumes:
    sessions:
    networks:
```

Make sure you have the following haproxy configuration in `./haproxy/haproxy.conf`:
```
defaults
    mode http
    timeout connect 5000ms

frontend http-in
    bind *:8080

    acl host_direct hdr_reg(host) -i ^.*\.direct\..*?:?.*$

    use_backend l2 if host_direct

    default_backend pwd 

backend pwd
    server node1 pwd:3000

backend l2
    server node2 l2:443
```

Start the Docker daemon on your machine and run `docker pull franela/dind`. Run play-with-docker with `docker-compose up`. Then open http://localhost in your browser. 

If you want to ssh into your containers add the following line to your `/etc/hosts` file:

```
127.0.0.1   localhost direct.localhost
```

## Development

Start the Docker daemon on your machine and run `docker pull franela/dind`. 

1) Install go 1.7.1+ with `brew` on Mac or through a package manager.

2) Install [dep](https://github.com/golang/dep) and run `dep ensure` to pull dependencies

3) Start PWD as a container with docker-compose up.

5) Point to http://localhost and click "New Instance"

Notes:

* There is a hard-coded limit to 5 Docker playgrounds per session. After 4 hours sessions are deleted.
* If you want to override the DIND version or image then set the environmental variable i.e.
  `DIND_IMAGE=franela/docker<version>-rc:dind`. Take into account that you can't use standard `dind` images, only [franela](https://hub.docker.com/r/franela/) ones work.
  
### Port forwarding

In order for port forwarding to work correctly in development you need to make `*.localhost` to resolve to `127.0.0.1`. That way when you try to access to `pwd10-0-0-1-8080.host1.localhost`, then you're forwarded correctly to your local PWD server.

You can achieve this by setting up a `dnsmasq` server (you can run it in a docker container also) and adding the following configuration:

```
address=/localhost/127.0.0.1
```

Don't forget to change your computer default DNS to use the dnsmasq server to resolve.

### Building the dind image myself.

If you want to make changes to the `dind` image being used, make your changes to the `Dockerfile.dind` file and then build it using this command: `docker build --build-arg docker_storage_driver=vfs -f Dockerfile.dind -t franela/dind .` 

## FAQ

### How can I connect to a published port from the outside world?


If you need to access your services from outside, use the following URL pattern `http://ip<hyphen-ip>-<session_jd>-<port>.direct.labs.play-with-docker.com` (i.e: http://ip-2-135-3-b8ir6vbg5vr00095iil0-8080.direct.labs.play-with-docker.com).

### Why is PWD running in ports 80 and 443?, Can I change that?.

No, it needs to run on those ports for DNS resolve to work. Ideas or suggestions about how to improve this
are welcome
