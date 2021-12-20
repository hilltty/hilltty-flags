> **Ayuda a la traducción**  
> Si observa algún error o carencia en la traducción, le rogamos que lo comunique y lo corrija. Gracias.

## ¿Por qué no [Aikar flags](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/)?

Su recolección de basura se basa en el algoritmo G1. Como él dijo, el algoritmo es increíblemente estable pero es increíblemente lento para los estándares actuales. Al mismo tiempo, está enormemente anticuado, todo lo que implementaba era innovador en los días del JDK 8, pero ahora no lo es. De hecho, ¿por qué cambiar algo que funciona? Pues debería hacerlo.

Propongo sustituirlo por Shenandoah - se trata de un recolector de basura con un tiempo de pausa increíblemente corto, que es muy adecuado para nuestro juego favorito, a todos no nos gustan las congelaciones. Esto no afectó a la estabilidad durante todo el período de pruebas ininterrumpidas. No se identificó ni un solo problema.

## Flags

**Supported JDK assemblies:**

*Recomiendo usar OpenJDK 17*

- [x] OpenJDK 8+
- [x] Red Hat 8+
- [x] Amazon 11+
- [x] Azul 11+
- [x] AdoptOpenJDK 11+
- [ ] Oracle
- [ ] SAP

**Servidores compatibles:**

- [x] Vanilla
- [x] Bukkit, Spigot, Paper ...
- [x] Fabric
- [x] Forge

**Propiedades terminadas:**

```yml
java -jar -server -Xms6G -Xmx6G -XX:+UseLargePages -XX:LargePageSizeInBytes=2M -XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu -XX:+UseNUMA -XX:+AlwaysPreTouch -XX:-UseBiasedLocking -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 launcher-airplane.jar --nogui
```

> Advertencia `Option UseBiasedLocking was deprecated in version 15.0 and will likely be removed in a future release.` puede ser ignorado y utilizado con seguridad en su servidor, la bandera UseBiasedLocking hace su trabajo sin problemas.

**Y ahora vamos a analizar detenidamente qué es lo responsable de qué:**

 *-Xms6G* y *-Xmx6G*: establece los límites de uso de la memoria por su servidor de Minecraft, recomiendo no utilizar más de 12 GB para su servidor y siempre dejar 1 - 2 GB de memoria libre para el sistema.

 *-XX:+UseLargePages* y *-XX:LargePageSizeInBytes = 2M*: **sólo para usuarios avanzados**, permite utilizar grandes páginas de memoria registrada, acelera la velocidad de arranque y la capacidad de respuesta del servidor. Hagamos que Linux registre páginas por nosotros.  Añade esta línea a `/etc/sysctl.conf`:

```yml
vm.nr_hugepages = 3372
```

¿Cómo conseguimos este número?  Digamos que quiero registrar 6 GB de páginas grandes, para ello, divido 6 GB por 2.

```yml
6 * 1024 / 2 = 3072
```

A continuación, recomiendo dejar un espacio libre y añadir 300 a nuestro número.

```yml
3072 + 300 = 3372
```

A continuación, reiniciamos el sistema para aplicar los cambios. Puedes verificar que la memoria se ha registrado correctamente con el comando
 `grep -i hugepages /proc/meminfo`.

---
*-XX:+UnlockExperimentalVMOptions*: permite el uso de características experimentales.

*-XX:+UseShenandoahGC*: utilizar el proyecto Shenandoah como algoritmo de recogida de basura.

*-XX:ShenandoahGCMode=iu*: activar el modo experimental de nuestro ensamblador, es un espejo del modo SATB, lo que hará que el marcado sea menos conservador, especialmente en lo que respecta al acceso a los enlaces débiles.

---
*-XX:+UseNUMA*: Permite el intercalado NUMA en hosts con múltiples sockets, cuando se combina con AlwaysPreTouch, proporciona un mejor rendimiento que la configuración por defecto.  Se pueden encontrar más detalles sobre esta arquitectura [desde aquí](https://en.wikipedia.org/wiki/Non-uniform_memory_access).

*-XX:+AlwaysPreTouch*: El prerregistro de toda la memoria asignada a la vez, reduce los retrasos de entrada.

*-XX:-UseBiasedLocking*: Hay una compensación entre el ancho de banda del bloqueo ilimitado (sesgado) y los puntos seguros que hace la JVM para activarlos y desactivarlos según sea necesario. Para las cargas de trabajo centradas en la latencia, incluidos los servidores de Minecraft, tiene sentido desactivar el bloqueo sesgado.

*-XX:+DisableExplicitGC*: Llamar a System.gc() desde el código personalizado obliga a ShenandoahGC a realizar un ciclo adicional de recolección de basura, lo que deshabilita la protección contra el código que abusa de ella.

## Software de servidor (núcleo)

Para la opción más estable y eficiente, yo recomendaría [Airplane](https://github.com/TECHNOVE/Airplane).

## System

Tuned-adm es una herramienta de línea de comandos que permite cambiar entre perfiles ajustados para mejorar el rendimiento en muchos casos de uso específicos.  Instale el paquete con `apt-get`:

```yml
sudo apt-get install tuned
```

A continuación, tienes que elegir la configuración para tu sistema, te recomiendo usar `throughput-performance` o `latency-performance`, establece el perfil que necesites:

```yml
sudo tuned-adm profile throughput-performance
```

Puede verificar que los cambios se han aplicado con el comando `tuned-adm profile`.

Un artículo detallado sobre todos los perfiles y cuándo utilizarlos [aquí](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-tool_reference-tuned_adm).

## Configuración adicional

### bukkit.yml

```yml
chunk-gc:
 period-in-ticks: 600
```

**Valor recomendado para `chunk-gc.period-in-ticks`:**  
No asigne más de 12 GB de memoria, esto no afectará a la mayoría de los casos.
| Memory / Number of players | up to 30 | 30 - 60 | 60 - 100 | over 100 |
| :--- | :---: | :---: | :---: | :---: |
| 4 GB | 400 | - | - | - |
| 8 GB | 600 | 400 | 300 | - |
| 12 GB | 1200 | 800 | 600 | 400 |
