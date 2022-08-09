# Docker Swarm Network MTU

<p align="center">
<img width="250px" src="https://memegenerator.net/img/instances/37438597.jpg">
</p>

If your Docker instance is communicating to other Docker instances via VXLAN or any other network that has a different MTU than the default 1500, you need to delete the default ingress network and create a new one! This is needed because Docker doesn't inherit the MTU of your networking interface, there will be intermittent packet losses when communicating between nodes!

Docker by default doesn't detect the network MTU because [reasons](https://github.com/moby/moby/pull/18108), so by default it use 1500 MTU for connections, which will cause issues if your network has a different MTU!

<p align="center">
<img width="100%" src="https://user-images.githubusercontent.com/9496359/183776273-14ff3146-f3e1-418f-a5ba-e4dd280666f3.png">
</p>

To fix this, check what is the MTU of the network that is going to be used for node communication by using `ip a`.

```
3: ens19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc fq_codel state UP group default qlen 1000
    link/ether ...
```

As we can see, our MTU is 1450, so let's setup Docker to use 1450 for network!

Create the file `/etc/docker/daemon.json` and insert the following contents, if the file already exists, add the `mtu` field to the already existing JSON.

```json
{
  "mtu": 1450
}
```

And then restart Docker with `systemctl restart docker`, this will setup Docker default networks to use 1450 MTU.

Now, we need to change the MTU of the ingress network used by Docker Swarm for inter node communication, to do this, you need to recreate the ingress network by doing this:
* `docker network rm ingress`
* `docker network create --driver overlay --ingress --opt com.docker.network.driver.mtu=MTUOfTheNetworkHere --subnet 192.168.128.0/24 --gateway 192.168.128.1 ingress`
  * To check what `subnet` and `gateway` you should use, use `docker inspect overlay` and check the IPAM section!
  * Let's suppose that we are using a VXLAN network for communication, VXLAN networks use a MTU size of 1450, so we need to use `--opt com.docker.network.driver.mtu=1450`
* Restart Docker with `systemctl restart docker`
 
This fixes Outside World -> Swarm Cluster communication, now we need to fix communication between your containers!

To do this, append this to the end of every `docker-compose.yml`.
 
```yml
networks:
  default:
    driver: overlay
    driver_opts:
      com.docker.network.driver.mtu: 1450
```

This will setup the overlay networks used by the stacks to also use your MTU.

And that's it! Now you shouldn't have any connectivity issues in your Swarm.

[[If you want to learn more, this issue talks a lot about connectivity issues!]](https://github.com/moby/moby/issues/36689#issuecomment-987706496)
