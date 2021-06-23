## Attention!
**[The article](https://github.com/hilltty/hilltty-flags/blob/main/russian-lang.md) was translated into English using Google Translate, if you are a native speaker please help with translation**

## Why not [Aikar flags](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/)?
It's very simple. His garbage collection is based on the G1 algorithm. As he said, the algorithm is incredibly stable, but it is extremely slow by current standards. At the same time, it is hugely outdated, everything that it implemented was innovative in the days of JDK 8, but now it is not. Indeed, why change something that works? Well, you should.

I propose to replace it with Shenandoah - this is a garbage collector with an extremely short pause time, which is so suitable for our favorite game, we all do not like freezes.  This did not affect stability in any way, during the entire period of uninterrupted testing, not a single problem was identified.
## Denial of responsibility
I do not urge everyone to immediately change their server launch properties, I just make it clear that nothing is perfect.  Also, I am not responsible for the stability of my parameters in your particular case, all systems are different, and the results are absolutely individual.
## Flags
**Supported JDK assemblies:**

*I recommend using OpenJDK 16*
- [x] OpenJDK 8+
- [x] Red Hat 8+
- [x] Amazon 11+
- [x] Azul 11+
- [x] AdoptOpenJDK 11+
- [ ] Oracle
- [ ] SAP

**Supported servers:**
- [x] Vanilla
- [x] Bukkit, Spigot, Paper ...
- [x] Fabric
- [x] Forge

**Finished properties:**
```yml
java -jar -server -Xms6G -Xmx6G -XX:+UseLargePages -XX:LargePageSizeInBytes=2M -XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu -XX:+UseNUMA -XX:+AlwaysPreTouch -XX:-UseBiasedLocking -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 launcher-airplane.jar --nogui
```
> Warning `Option UseBiasedLocking was deprecated in version 15.0 and will likely be removed in a future release.` can safely be ignored and used on your server, the UseBiasedLocking flag does its job just fine.

**And now we will carefully analyze what is responsible for what:**

 *-Xms6G* and *-Xmx6G*: sets the limits of memory usage by your Minecraft server, I recommend not using more than 12 GB for your server and always leave 1 - 2 GB of free memory for the system.

 *-XX:+UseLargePages* and *-XX:LargePageSizeInBytes = 2M*: **for advanced users only**, allows large pages of registered memory to be used, accelerates startup speed and server responsiveness.  Let's get Linux to register pages for us.  Add this line to `/etc/sysctl.conf`:
```yml
vm.nr_hugepages = 3372
```
How did we get this number?  Let's say I want to register 6 GB of large pages, for this I divide 6 GB by 2.
```yml
6 * 1024 / 2 = 3072
```
Next, I recommend leaving some free space, and adding 300 to our number.
```yml
3072 + 300 = 3372
```
Then we reboot the system to apply the changes. You can verify that the memory has been successfully registered with the command `grep -i hugepages /proc/meminfo`.

---
*-XX:+UnlockExperimentalVMOptions*: enables the use of experimental features.

*-XX:+UseShenandoahGC*: use the Shenandoah project as a garbage collection algorithm (this is how the translator reads this name).

*-XX:ShenandoahGCMode=iu*: turn on the experimental mode of our assembler, it is a mirror of the SATB mode, which will make the markup less conservative, especially regarding access to weak links.

---
*-XX:+UseNUMA*: Enables NUMA interleaving on hosts with multiple sockets, when combined with AlwaysPreTouch, it provides better performance than the default out-of-box configuration.  More details about this architecture can be found [from here](https://en.wikipedia.org/wiki/Non-uniform_memory_access).

*-XX:+AlwaysPreTouch*: pre-registration of all allocated memory at once, reduces input delays.

*-XX:-UseBiasedLocking*: There is a trade-off between the bandwidth of unlimited (biased) locking and the safe points the JVM makes to turn them on and off as needed. For latency-focused workloads, including Minecraft server, it makes sense to disable biased blocking.

*-XX:+DisableExplicitGC*: Calling System.gc () from custom code forces ShenandoahGC to perform an additional garbage collection cycle, disabling protects against code abusing it.
## Server software (core)
For the most stable and efficient option, I would recommend [Airplane](https://github.com/TECHNOVE/Airplane).
## System
Tuned-adm is a command line tool that allows you to switch between tuned profiles to improve performance in a number of specific use cases.  Install the package with `apt-get`:
```yml
sudo apt-get install tuned
```
Next, you need to choose the config for your system, I recommend using `throughput-performance` or` latency-performance`, set the profile you need:
```yml
sudo tuned-adm profile throughput-performance
```
You can verify that the changes have been applied with the command `tuned-adm profile`.

A detailed article about all profiles and when to use them [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-tool_referencem-tuning).
## Additional configuration
### bukkit.yml
```yml
chunk-gc:
 period-in-ticks: 600
```
**Recommended value for `chunk-gc.period-in-ticks`:**
Do not allocate more than 12 GB of memory, this will have no effect in most cases.
| Memory / Number of players | up to 30 | 30 - 60 | 60 - 100 | over 100 |
| :--- | :---: | :---: | :---: | :---: |
| 4 GB | 400 | - | - | - |
| 8 GB | 600 | 400 | 300 | - |
| 12 GB | 1200 | 800 | 600 | 400 |
 ### spigot.yml
 ```yml
 world-settings:
  default:
   max-tick-time:
    tile: 10
    entity: 20
```
## That's all for now
This page is still under development, so stay tuned :)
