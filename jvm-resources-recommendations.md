# JVM Resources Recommendations

Messing with `limits` and `reservations` may impact your Java application in ways that you weren't expecting, so here's some tips to not fail.

Don't worry, *it is also painfully hard on Kubernetes too*. 😭

My Swarm node Virtual Machine has 4GBs of RAM, 4 cores. So let's do some tests on it! The application will be using Java 17 (you MUST use Java 8.0_131 or above because Java didn't respect cgroups before that. We are in `${InsertYearHere}` already, move on to Java 17!!) and will print memory and CPU stats to the console before quitting. If you want to play around with it on your computer, [the container is public](https://github.com/MrPowerGamerBR/DebugAllocationContainers/pkgs/container/debugallocationcontainers)!
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

## No resources set

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

## With Xmx/Xms set
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

## Xmx/Xms + Resource Reservations (The Best And Simplest Way Of Doing This™)
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

## But what if we don't set `Xmx/Xms` WHILE we have a `reservations` set?
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

## But what if we don't set `Xmx/Xms` WHILE we have a `limits` set?
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

## Should I limit CPU?

In my opinion? Nah, let your services use all of your CPUs. Less headaches.

## But should I create CPU reservations?

Yes! Mostly to avoid scheduling your containers in oversatured nodes.

## JVM Resources tl;dr:

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
