<p align="center">
<img width="100%" src="https://user-images.githubusercontent.com/9496359/183309009-9268544a-b47b-44ef-8933-848e2f345aa2.png">
</p>

# Docker Swarm Mode Tutorial

I know, Kubernetes is what all the cool kidz are using nowadays! And Kubernetes is cool too... until it explodes and you spend hours trying to debug the issue. In fact, I decided to learn about Docker Swarm after getting burned by Kubernetes for the nth time... 

Kubernetes may be painless if you are using a managed Kubernetes instance, but if you are selfhosting it... it is hard!

**Let's face it:** You ain't *insert huge company name here*, you just care about hosting some containers' replicas, you don't care about all the fancy big corp stuff that Kubernetes tries to provide to you... because you ain't a big corp!

So why not use Docker Swarm?

## But is Docker Swarm... good?

If you are just like me that likes to scour the internet to see if users think that *insert tech here* is good, here are some posts on the internet about people talking about their experience with Docker Swarm!

* https://blog.kronis.dev/articles/docker-swarm-over-kubernetes
* https://news.ycombinator.com/item?id=29448182
* https://news.ycombinator.com/item?id=32306857
* https://news.ycombinator.com/item?id=32305800
* https://news.ycombinator.com/item?id=32305952
* https://news.ycombinator.com/item?id=32307229

## When SHOULD I use Docker Swarm?

Docker Swarm is a orchestrator, the "best little multi node orchestrator that could". It is very useful if you are already using Docker and you need to replicate your containers in multiple nodes/VMs.

* If you are self hosting your services in a dedicated server or in a VPS. (Example: dedicated servers @ OVH, Hetzner, etc)
* If you are already using Docker Compose.
  * Docker Swarm uses Docker Compose files for deployment too, so it is a natural step from Docker Compose -> Swarm! And I mean it! You just need to add a new `deploy` key to your already existing compose files to setup how do you want your container to be replicated and deployed... and that's it!
* If your services don't require persistent storage.

## When SHOULDN'T I use Docker Swarm?

As with any tool, there are some disadvantages to Docker Swarm, here are some reasons that you may want to avoid using Docker Swarm:

* You are using AWS/GCP/Azure/etc.
  * Just use their managed Kubernetes service. As I said before: Kubernetes is cool, except when it breaks. However using a Kubernetes instance that is managed by someone else lifts all the heavyweight from you, in exchange of a pretty hefty price increase.
* You need to auto scale according to demand.
  * However if you are using a rented dedicated server, why would tou want to auto scale? And if you are using a host that charges per usage (like AWS, GCP, Azure, etc), then they probably has a managed Kubernetes instance.
* You need to have distributed storage because you want to have a distributed database.
  * Docker Swarm does support distributed storage, however Kubernetes' solutions is more rock solid so just go with Kubernetes. But then again, do you *really* need a distributed storage for your smol service? You can go pretty far with a single instance database!
* You need a feature that isn't supported by Docker Swarm.
  * While Docker Swarm isn't deprecated, it is also not actively developed, so if you require a feature that isn't supported by it, it may take a loooong time until it is implemented.
* You like to have shiny buzzwords in your CV.

## But should I use Docker Swarm?

Contrary to the popular belief, Docker Swarm is not deprecated! Yeah, sure, sadly there isn't too much dev work on Docker Swarm nowadays, but it isn't *dead* or *deprecated*, and after all, if it fits your needs, does it really need to be actively worked on? As long as it works, and if you get to a point where Docker Swarm is not fitting your needs, *then* move to Kubernetes! Don't overcomplicate your life right now just because some day, *maybe*, you would need Kubernetes.

If it does get deprecated, you can migrate off of it by converting your Docker Compose deployment files with [Kompose](https://kompose.io/), so it is not like you are going to end up being stuck on a dead platform forever.

## Official Tutorial

Docker has a Swarm Mode tutorial on their website, [check it out](https://docs.docker.com/engine/swarm/swarm-tutorial/)!

Docker also has a tutorial on how to deploy Docker Compose files (also known as "Stack") to Docker Swarm, [check it out](https://docs.docker.com/engine/swarm/stack-deploy/)!

This tutorial is mostly a "I'm writing things as I'm learning", focused more on the Docker Swarm + Docker Compose combo than the official tutorial's Docker Swarm + Docker CLI combo, so there may be things that are incorrect or misleading, however I think that other people may think that this tutorial is also useful!

## Install Ubuntu

I'm using a Ubuntu 22.04 VM for this tutorial.

## Install Docker Engine

https://docs.docker.com/get-docker/

## Start Docker Swarm Mode

Start your Docker Swarm with `docker swarm init`!

If you have multiple network interfaces, Docker will ask you to choose what IP to use. You need to choose the IP that can access your other Docker instances!

By default, Docker will use IPs on the 10.0.0.0 range, if your machine's interface is behind NAT and it also uses the 10.0.0.0 range, it will cause issues when you try to access containers hosted in your Docker Swarm!

To fix this, initialize your Docker Swarm with the `--default-addr-pool` parameter! Let's suppose we want to use `192.168.128.0/18` for our containers. [[🐳 Learn more]](https://docs.docker.com/engine/swarm/swarm-mode/#configuring-default-address-pools)

```bash
sudo docker swarm init --default-addr-pool 192.168.128.0/18
```

**Attention:** If you change your `default-addr-pool`, check if the IPs aren't being used by another network! `ip a` shows what IP ranges are being used. If you use an IP range that is already being used by something else, your containers won't start due to `Pool overlaps with other one on this address space`!

If you get `Swarm initialized: current node (qgsfyhmhwtpkp7zpo7lts2vhp) is now a manager.`, then your Docker instance is in a swarm, and your node is a manager node, sweet!

**Attention:** If you have multiple network interfaces, Docker will say `Error response from daemon: could not choose an IP address to advertise since this system has multiple addresses on different interfaces (10.29.10.1 on ens18 and 172.29.10.1 on ens19) - specify one with --advertise-addr`. In this case, the `--advertise-addr` parameter should be the IP that *can communicate with other nodes*! So, if `172.29.10.1` is the IP that can access other nodes, then that should be your `--advertise-addr`.

**Attention:** If your Docker instance is communicating to other Docker instances via VXLAN or any other network that has a lower MTU than 1500, you need to delete the default ingress network and create a new one! This is needed because Docker also uses VXLAN, so if you don't change the MTU, there will be intermittent packet losses when communicating between nodes! More information at https://github.com/moby/moby/issues/36689#issuecomment-987706496
* If you are using a VXLAN network for node communication (which Docker will use that network to use VXLAN on top of it), you need to decrease your MTU from the default 1450 to 1400
  * `docker network rm ingress`
  * `docker network create --driver overlay --ingress --opt com.docker.network.driver.mtu=1400 --subnet 192.168.128.0/24 --gateway 192.168.128.1 ingress`
    * To check what `subnet` and `gateway` you should use, use `docker inspect overlay` and check the IPAM section!
  * Restart Docker with `systemctl restart docker`

## Accessing private images hosted on GitHub's `ghcr.io`

While not related to Docker Swarm, I thought it was nice to talk about this. :3

```bash
docker login ghcr.io
```

Use your GitHub username as your Username, and a [Personal Access Token](https://github.com/settings/tokens) as your Password.

**Attention:** You will need to login with your account on every node in the Docker Swarm! If you don't, some nodes may fail to run your container because they aren't able to download your container image!

## Hosting your first service

Did you know that Docker Swarm uses `docker-compose.yml` files??? Crazy huh?

```yaml
version: '3.9'
services:
    helloworld:
        # This will set the hostname to helloworld-ReplicaID
        hostname: "helloworld-{{.Task.Slot}}"
        # The image, we will use a helloworld http image
        image: strm/helloworld-http
        # We will expose the service at port 8080 on the host
        ports:
            - "8080:80"
        # Docker Swarm configuration deployment configurations!
        deploy:
            # We want to replicate our service...
            mode: replicated
            # And it will have two instances of the container!
            replicas: 2
```

Now let's deploy our service with `docker stack deploy --compose-file docker-compose.yml stackdemo`!

If everything goes well, you will be able to see your stack on the `docker stack ls` command!
```bash
mrpowergamerbr@docker-swarm-test:~$ sudo docker stack ls
NAME        SERVICES   ORCHESTRATOR
stackdemo   1          Swarm
```

And you can also see the service status with `docker stack services stackdemo`
```bash
mrpowergamerbr@docker-swarm-test:~$ sudo docker stack services stackdemo
ID             NAME              MODE         REPLICAS   IMAGE                         PORTS
totx5zzra290   stackdemo_helloworld   replicated   2/2        strm/helloworld-http:latest   *:8080->80/tcp
```

You can also view events about the stack with `docker stack ps stackdemo`
```bash
swarm@docker-swarm-manager-1:~$ sudo docker stack ps powercms
ID             NAME                      IMAGE                                                                                                     NODE                     DESIRED STATE   CURRENT STATE             ERROR                              PORTS
ztf65gnt1fjc   powercms_powercms.1       ghcr.io/mrpowergamerbr/powercms@sha256:41e5cf391194792b4ddff8d0a5561122213cb793ada75edc1e4e6a5ba9b90e16   docker-swarm-manager-1   Ready           Rejected 4 seconds ago    "Pool overlaps with other one …"
9oiwb29bc1eu    \_ powercms_powercms.1   ghcr.io/mrpowergamerbr/powercms@sha256:41e5cf391194792b4ddff8d0a5561122213cb793ada75edc1e4e6a5ba9b90e16   docker-swarm-manager-1   Shutdown        Rejected 9 seconds ago    "Pool overlaps with other one …"
ezif348rn177    \_ powercms_powercms.1   ghcr.io/mrpowergamerbr/powercms@sha256:41e5cf391194792b4ddff8d0a5561122213cb793ada75edc1e4e6a5ba9b90e16   docker-swarm-worker-1    Shutdown        Rejected 14 seconds ago   "Pool overlaps with other one …"
xszs8mbxrzdh    \_ powercms_powercms.1   ghcr.io/mrpowergamerbr/powercms@sha256:41e5cf391194792b4ddff8d0a5561122213cb793ada75edc1e4e6a5ba9b90e16   docker-swarm-worker-1    Shutdown        Rejected 19 seconds ago   "Pool overlaps with other one …"
nnq6i1ybb4v4    \_ powercms_powercms.1   ghcr.io/mrpowergamerbr/powercms@sha256:41e5cf391194792b4ddff8d0a5561122213cb793ada75edc1e4e6a5ba9b90e16   docker-swarm-manager-1   Shutdown        Rejected 24 seconds ago   "Pool overlaps with other one …"
```

Let's also check the containers running on our Docker instance...

```bash
mrpowergamerbr@docker-swarm-test:~$ sudo docker ps
CONTAINER ID   IMAGE                         COMMAND      CREATED          STATUS          PORTS     NAMES
25088b353de7   strm/helloworld-http:latest   "/main.sh"   41 minutes ago   Up 41 minutes   80/tcp    stackdemo_front.2.wvzs76sk9iq05xozgz4x9cdqe
18588ab9c07c   strm/helloworld-http:latest   "/main.sh"   41 minutes ago   Up 41 minutes   80/tcp    stackdemo_front.1.nmreutrz6q4l57rolejuco6xa
```

**Attention:** `docker ps` only shows the containers running on the current Docker instance! Don't forget about this if you include multiple Docker instances on the swarm!

## Accessing services

Now let's access the service with `curl`!
```bash
mrpowergamerbr@docker-swarm-test:~$ curl 127.0.0.1:8080
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from helloworld-2</h1></body></html
```

Now, what happens if we do a bunch of requests?
```bash
mrpowergamerbr@docker-swarm-test:~$ curl 127.0.0.1:8080
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from helloworld-1</h1></body></html
mrpowergamerbr@docker-swarm-test:~$ curl 127.0.0.1:8080
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from helloworld-2</h1></body></html
mrpowergamerbr@docker-swarm-test:~$ curl 127.0.0.1:8080
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from helloworld-1</h1></body></html
mrpowergamerbr@docker-swarm-test:~$ curl 127.0.0.1:8080
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from helloworld-2</h1></body></html
```

As you can see, the hostname is alternating between `helloworld-1` and `helloworld-2`, showing that Docker Swarm is load balancing between your two `helloworld-http` replicas!

From within any of the containers of the same stack, you can access the other instance by its hostname!
```bash
mrpowergamerbr@docker-swarm-test:~$ sudo docker exec -it e4999d8d5cb9 /bin/bash
root@helloworld-1:/www# ping helloworld-2
PING helloworld-2 (192.168.2.4): 56 data bytes
64 bytes from 192.168.2.4: icmp_seq=0 ttl=64 time=0.093 ms
64 bytes from 192.168.2.4: icmp_seq=1 ttl=64 time=0.073 ms
64 bytes from 192.168.2.4: icmp_seq=2 ttl=64 time=0.071 ms
^C--- helloworld-2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.071/0.079/0.093/0.000 ms
root@helloworld-1:/www# apt install curl
... installing curl ...
root@helloworld-1:/www# curl helloworld-2
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from helloworld-2</h1></body></html
root@helloworld-1:/www# curl helloworld-2
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from helloworld-2</h1></body></html
root@helloworld-1:/www# curl helloworld-2
<html><head><title>HTTP Hello World</title></head><body><h1>Hello from helloworld-2</h1></body></html
```

You can also access the service externally at `YourMachineIP:8080`!

## Applying updates to your stack
You can apply updates to your stack with `docker stack deploy --compose-file docker-compose.yml stackdemo`!

**Attention:** If you had an service, removed it from the Compose file and used `stack deploy`, Docker will *not* remove the already running services! To remove an service, use `sudo docker service rm servicename`, you can see all of your stack's running services with `sudo docker stack services stackdemo`!

TODO: Rolling Updates

## Application Configurations
```bash
mrpowergamerbr@docker-swarm-test:~$ echo "Loritta is so cute! :3" > loritta.txt
```

```yaml
version: '3.9'
services:
    helloworld:
        hostname: "helloworld-{{.Task.Slot}}"
        image: strm/helloworld-http
        ports:
            - "8080:80"
        deploy:
            mode: replicated
            replicas: 2
        # The configuration!
        configs:
            - source: my_first_config # The source should match the config name in the "configs" section
              target: /loritta_cute.txt # Where the config should be mounted

configs:
    my_first_config:
        file: ./loritta.txt
```

`docker stack deploy --compose-file docker-compose.yml stackdemo`

```bash
mrpowergamerbr@docker-swarm-test:~$ sudo docker exec -it 66726bffdd6d /bin/bash
root@helloworld-2:/www# cat /loritta_cute.txt
Loritta is so cute! :3
```

**Attention:** If you update your configuration file, the `docker stack deploy --compose-file docker-compose.yml stackdemo` command will fail! This is because there is another container that is already using the configuration file. To workaround this issue, change the configuration name (`my_first_config`) after changing anything in the configuration file!
* **Idea:** Suffix the file's hash at the end of the configuration file, that's what Kubernetes' Kustomize does, and then create a script that periodically tries to delete all configs from your Docker Swarm cluster, configs that are in use won't be deleted by Docker. :P

## Limiting Resources for the Scheduler

Just like Kubernetes, you can also set resources requests and limits to your application!

If you already used Kubernetes before...
* k8s' `limits` = Swarm's `limits`
* k8s' `requests` = Swarm's `reservations`

* `reservations`: When scheduling a container, Swarm MUST guarantee container can allocate at least the configured amount
  * If your container is configured to `reservations.memory: 4G`, and none of your Swarm nodes have 4GB+ of RAM available, the node won't be scheduled due to `insufficient resources on X nodes`!
* `limits`: When scheduling a container, Swarm MUST prevent container to allocate more
  * If your container is configured to `limits.memory: 4G`, and your container is using more than 4GBs of RAM, Swarm will terminate your container automatically!

You can set both, only one of them, or none! Same goes for the resources specified within the section.

[[🐳 Learn more]](https://docs.docker.com/compose/compose-file/deploy/#resources)

```yaml
version: '3.9'
services:
    helloworld:
        hostname: "helloworld-{{.Task.Slot}}"
        image: strm/helloworld-http
        ports:
            - "8080:80"
        deploy:
            mode: replicated
            replicas: 2
            resources:
                limits:
                    cpus: '0.50'
                    memory: 50M
                reservations:
                    cpus: '0.25'
                    memory: 4G
```

### JVM Resources Recommendations

Messing with `limits` and `reservations` may impact your Java application in ways that you weren't expecting, so here's some tips to not fail.

Don't worry, *it is also painfully hard on Kubernetes too*. 😭

My Swarm node Virtual Machine has 4GBs of RAM, 4 cores. So let's do some tests on it! The application will be using Java 17 (you MUST use Java 8.0_131 or above because Java didn't respect cgroups before that. We are in `${InsertYearHere}` already, move on to Java 17!!) The code that we will be using is:
```kotlin
fun main() {
    val mb = 1024 * 1024
    val runtime = Runtime.getRuntime()

    println("Used Memory: ${(runtime.totalMemory() - runtime.freeMemory()) / mb}MiB")
    println("Free Memory: ${runtime.freeMemory() / mb}MiB")
    println("Total Memory: ${runtime.totalMemory() / mb}MiB")
    println("Max Memory: ${runtime.maxMemory() / mb}MiB")
    println("Available Processors: ${Runtime.getRuntime().availableProcessors()}")
}
```

#### No resources set

```yml
version: "3.9"
services:
  temurin:
    image: ghcr.io/mrpowergamerbr/debugallocationcontainers@sha256:d98ad5df3b5829fc7595eb48f6e49c9856cd9ad8ebefe75068ecd5063f0fb789
    environment:
      JAVA_TOOL_OPTIONS: "-verbose:gc"
```
**Output:**
```
temurin-test_temurin.1.xnwoo9oyyhfx@docker-swarm-worker-1    | [0.005s][info][gc] Using G1
temurin-test_temurin.1.xnwoo9oyyhfx@docker-swarm-worker-1    | Used Memory: 1MiB
temurin-test_temurin.1.xnwoo9oyyhfx@docker-swarm-worker-1    | Free Memory: 62MiB
temurin-test_temurin.1.xnwoo9oyyhfx@docker-swarm-worker-1    | Total Memory: 64MiB
temurin-test_temurin.1.xnwoo9oyyhfx@docker-swarm-worker-1    | Max Memory: 982MiB
temurin-test_temurin.1.xnwoo9oyyhfx@docker-swarm-worker-1    | Available Processors: 4
```

The JVM memory is set to 1/4 of the entire VM. This makes sense, because the default value of `-XX:MaxRAMPercentage` is 25, and 25% of 4GBs is 1GB.

#### With Xmx/Xms set
Most Java developers use `-Xmx` and `-Xms` to set the heap size, so let's use it.

```yml
version: "3.9"
services:
  temurin:
    image: ghcr.io/mrpowergamerbr/debugallocationcontainers@sha256:d98ad5df3b5829fc7595eb48f6e49c9856cd9ad8ebefe75068ecd5063f0fb789
    environment:
      JAVA_TOOL_OPTIONS: "-verbose:gc -Xmx512M -Xms512M"
```
**Output:**
```
temurin-test_temurin.1.ulbt33xyo0d0@docker-swarm-worker-1    | [0.005s][info][gc] Using G1
temurin-test_temurin.1.ulbt33xyo0d0@docker-swarm-worker-1    | Used Memory: 1MiB
temurin-test_temurin.1.ulbt33xyo0d0@docker-swarm-worker-1    | Free Memory: 510MiB
temurin-test_temurin.1.ulbt33xyo0d0@docker-swarm-worker-1    | Total Memory: 512MiB
temurin-test_temurin.1.ulbt33xyo0d0@docker-swarm-worker-1    | Max Memory: 512MiB
temurin-test_temurin.1.ulbt33xyo0d0@docker-swarm-worker-1    | Available Processors: 4
```

As we can see, the JVM is using our configured allocated memory! However, Swarm doesn't know about this, so *it will try to allocate our container on any node, even if they don't have 512MB available!*

#### Xmx/Xms + Resource Reservations (The Best And Simplest Way Of Doing This™)
```yml
version: "3.9"
services:
  temurin:
    image: ghcr.io/mrpowergamerbr/debugallocationcontainers@sha256:d98ad5df3b5829fc7595eb48f6e49c9856cd9ad8ebefe75068ecd5063f0fb789
    deploy:
      resources:
        reservations:
          memory: 768M # We reserve more memory than we set the heap, due to off heap allocations and other JVM shenanigans.
    environment:
      JAVA_TOOL_OPTIONS: "-verbose:gc -Xmx512M -Xms512M"
```
**Output:**
```
temurin-test_temurin.1.vrp8yjosnc5d@docker-swarm-manager-1    | [0.005s][info][gc] Using G1
temurin-test_temurin.1.vrp8yjosnc5d@docker-swarm-manager-1    | Used Memory: 1MiB
temurin-test_temurin.1.vrp8yjosnc5d@docker-swarm-manager-1    | Free Memory: 510MiB
temurin-test_temurin.1.vrp8yjosnc5d@docker-swarm-manager-1    | Total Memory: 512MiB
temurin-test_temurin.1.vrp8yjosnc5d@docker-swarm-manager-1    | Max Memory: 512MiB
temurin-test_temurin.1.vrp8yjosnc5d@docker-swarm-manager-1    | Available Processors: 4
```

Once again, it works fine! In my opinion, this is the best way AND easiest way to do this.

#### But what if we don't set `Xmx/Xms` WHILE we have a `reservations` set?
```yml
version: "3.9"
services:
  temurin:
    image: ghcr.io/mrpowergamerbr/debugallocationcontainers@sha256:d98ad5df3b5829fc7595eb48f6e49c9856cd9ad8ebefe75068ecd5063f0fb789
    deploy:
      resources:
        reservations:
          memory: 768M # We reserve more memory than we set the heap, due to off heap allocations and other JVM shenanigans.
    environment:
      JAVA_TOOL_OPTIONS: "-verbose:gc"
```
**Output:**
```
temurin-test_temurin.1.wost4c51s6ma@docker-swarm-manager-1    | [0.005s][info][gc] Using G1
temurin-test_temurin.1.wost4c51s6ma@docker-swarm-manager-1    | Used Memory: 1MiB
temurin-test_temurin.1.wost4c51s6ma@docker-swarm-manager-1    | Free Memory: 62MiB
temurin-test_temurin.1.wost4c51s6ma@docker-swarm-manager-1    | Total Memory: 64MiB
temurin-test_temurin.1.wost4c51s6ma@docker-swarm-manager-1    | Max Memory: 984MiB
temurin-test_temurin.1.wost4c51s6ma@docker-swarm-manager-1    | Available Processors: 4
```

It still allocates ~1GBs, so as you can see the container just ignores our `reservations` when figuring out how much memory it can allocate.

#### But what if we don't set `Xmx/Xms` WHILE we have a `limits` set?
```yml
version: "3.9"
services:
  temurin:
    image: ghcr.io/mrpowergamerbr/debugallocationcontainers@sha256:d98ad5df3b5829fc7595eb48f6e49c9856cd9ad8ebefe75068ecd5063f0fb789
    deploy:
      resources:
        limits:
          memory: 512M
    environment:
      JAVA_TOOL_OPTIONS: "-verbose:gc"
```
**Output:**
```
temurin-test_temurin.1.vq0a6qz6kjkr@docker-swarm-manager-1    | [0.002s][info][gc] Using Serial
temurin-test_temurin.1.vq0a6qz6kjkr@docker-swarm-manager-1    | Used Memory: 0MiB
temurin-test_temurin.1.vq0a6qz6kjkr@docker-swarm-manager-1    | Free Memory: 7MiB
temurin-test_temurin.1.vq0a6qz6kjkr@docker-swarm-manager-1    | Total Memory: 7MiB
temurin-test_temurin.1.vq0a6qz6kjkr@docker-swarm-manager-1    | Max Memory: 123MiB
temurin-test_temurin.1.vq0a6qz6kjkr@docker-swarm-manager-1    | Available Processors: 4
```

Ah ha! Now the JVM is using 25% of the memory that we set on the `limits` section! And look, because our heap is smol, Java decides that using Serial instead of G1GC is a good idea (Spoiler: While Serial is good for desktop applications that don't have a lot of threads, it isn't good for anything else).

Now, you can use `-XX:MaxRAMPercentage` to set how much % your JVM should allocate of the `limits.memory` that you have set. While this works, I do think that this is a bit confusing and non-intuitive, and besides, 99% of the times you are deploying containers that have the same max memory set, so this is not that useful.

#### Should I limit CPU?

In my opinion? Nah, let your services use all of your CPUs. Less headaches.

#### But should I create CPU reservations?

Yes! Mostly to avoid scheduling your containers in oversatured nodes.

#### JVM Resources tl;dr:

* Create memory reservations to avoid allocating your container to a node that doesn't have enough memory. Always reserve a bit more memory than what you are allocating to avoid the system killing your JVM app due to low memory.
* Create CPU reservations to avoid allocating your container to a node that is oversaturated.
* Set your memory with `-Xmx` and `-Xms` because it is easier than fiddling with `MaxRAMPercentage`.

```yml
version: "3.9"
services:
  temurin:
    image: ghcr.io/mrpowergamerbr/debugallocationcontainers@sha256:d98ad5df3b5829fc7595eb48f6e49c9856cd9ad8ebefe75068ecd5063f0fb789
    deploy:
      resources:
        reservations:
          memory: 768M
          cpus: "0.5"
    environment:
      JAVA_TOOL_OPTIONS: "-verbose:gc -Xmx512M -Xms512M"
```

## Deleting your stack
You can delete the stack with `docker stack rm stackdemo`, this will delete the stack + service + all containers associated with it!

```bash
mrpowergamerbr@docker-swarm-test:~$ sudo docker stack rm stackdemo
Removing service stackdemo_helloworld
Removing network stackdemo_default
```

## Adding more nodes to your Swarm

Having only one Docker instance in your swarm is kinda pointless, the fun really begins after we add other Docker instances to our swarm! This way, replicas can be hosted in different nodes, and this is all handled by Docker Swarm.

`sudo docker swarm join-token worker`

Copy the command and execute it on your VM!

`docker swarm join --token SWMTKN-1-2s4fg0ctnjcgy2hei784kz1c3lmc15axeozlfgs08obm6cajzj-7ycl00ssan3glz3309ac79kl4 10.0.12.10:2377`

If you get `This node joined a swarm as a worker.`, then it means that now your Docker instance is now in the Swarm!

On your manager node, use `docker node ls` to view all nodes on the swarm.

```
swarm@docker-swarm-manager-1:/etc/netplan$ sudo docker node ls
ID                            HOSTNAME                 STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
qgsfyhmhwtpkp7zpo7lts2vhp *   docker-swarm-manager-1   Ready     Active         Leader           20.10.17
d0o8mcxcjb4fwk8jxqxym0kcc     docker-swarm-worker-1    Ready     Active                          20.10.17
```

## Replicating in Multiple Nodes

TODO

## Load Balancing between your Docker Swarm nodes with nginx

You can access your service at `YourMachineIP:8080`, but what if I told you that there is *another way* to access your service?

You can access the service at *any* node IP! Not just your machine IP, and it can even be in a machine that *isn't* hosting your service at the moment! As long as it is connected to your swarm, you are able to access it.

If you ever used k3s' Load Balancer "Klipper", it works in the exact same way!

With this, we can load balance our service with nginx!

```conf
upstream powercms_backend {
    server 172.29.10.1:8080;
    server 172.29.11.1:8080;
}

server {
    listen 443 ssl;
    server_name mrpowergamerbr.com;

    include mrpowergamerbr_ssl.conf;

    location / {
        proxy_pass http://powercms_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Conclusion

TODO
