> **Aide à la traduction**  
> Si vous remarquez des erreurs ou des défauts dans la traduction, veuillez les signaler et les corriger. Merci !

## Pourquoi pas [Aikar flags](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/)?

C'est très simple. Son garbage collector est basé sur l'algorithme G1. Comme il l'a dit, l'algorithme est incroyablement stable, mais il est extrêmement lent par rapport aux normes actuelles. En même temps, il est terriblement dépassé, tout ce qu'il mettait en œuvre était innovant à l'époque du JDK 8, mais plus maintenant. En effet, pourquoi changer quelque chose qui fonctionne ? Eh bien, vous devriez.

Je propose de le remplacer par Shenandoah - il s'agit d'un garbage collector avec un temps de pause extrêmement court, ce qui convient parfaitement à notre jeu préféré, nous détestons tous les freezes.  Cela n'a en aucun cas affecté la stabilité, durant l'antière période de tests ininterrompus, pas un seul problème n'a été identifié.

## Déni de responsabilité

Je ne demande pas à tout le monde de modifier immédiatement les propriétés de lancement de leur serveur, je veux simplement préciser que rien n'est parfait. De même, je ne suis pas responsable de la stabilité de mes paramètres dans votre cas d'utilisation en particulier, tous les systèmes sont différents, et les résultats peuvent varier.

## Flags

**JDK supportés:**

*Je recommande d'utiliser OpenJDK 17*

- [x] OpenJDK 8+
- [x] Red Hat 8+
- [x] Amazon 11+
- [x] Azul 11+
- [x] AdoptOpenJDK 11+
- [ ] Oracle
- [ ] SAP

**Serveurs pris en charge:**

- [x] Vanilla
- [x] Bukkit, Spigot, Paper ...
- [x] Fabric
- [x] Forge

**Propriétés:**

```yml
java -jar -server -Xms6G -Xmx6G -XX:+UseLargePages -XX:LargePageSizeInBytes=2M -XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu -XX:+UseNUMA -XX:+AlwaysPreTouch -XX:-UseBiasedLocking -XX:+DisableExplicitGC -Dfile. encoding=UTF-8 launcher-airplane.jar --nogui
```

> L'avertissement `Option UseBiasedLocking was deprecated in version 15.0 and will likely be removed in a future release.` peut être ignoré sans risque et utilisé sur votre serveur, le flag UseBiasedLocking fait très bien son travail.

**Et maintenant nous allons analyser attentivement ce qui est responsable de quoi:**

 *-Xms6G* et *-Xmx6G* : définit les limites d'utilisation de la mémoire par votre serveur Minecraft, je recommande de ne pas utiliser plus de 12 Go pour votre serveur et de toujours laisser 1 - 2 Go de mémoire libre pour le système.

 *-XX:+UseLargePages* et *-XX:LargePageSizeInBytes = 2M* : **pour les utilisateurs avancés uniquement**, permet d'utiliser de large pages de mémoire, accélère la vitesse de démarrage et la réactivité du serveur. Allons faire en sorte que Linux enregistre des pages pour nous. Ajoutez cette ligne à `/etc/sysctl.conf` :

```yml
vm.nr_hugepages = 3372
```

Comment avons-nous obtenu ce nombre ? Disons que je veux enregistrer 6 Go de large pages, pour cela, je divise 6 Go par 2.

```yml
6 * 1024 / 2 = 3072
```

Ensuite, je recommande de laisser un peu d'espace libre et d'ajouter 300 à notre nombre.

```yml
3072 + 300 = 3372
```

Puis nous redémarrons le système pour appliquer les changements. Vous pouvez vérifier que la mémoire a été enregistrée avec succès avec la commande `grep -i hugepages /proc/meminfo`.

---
*-XX:+UnlockExperimentalVMOptions* : permet l'utilisation de fonctionnalités expérimentales.

*-XX:+UseShenandoahGC* : utilise le projet Shenandoah comme algorithme de garbage collector (c'est ainsi que le traducteur lit ce nom).

*-XX:ShenandoahGCMode=iu* : active le mode expérimental de notre assembleur, c'est un miroir du mode SATB, qui rendra le markup moins conservateur, surtout en ce qui concerne l'accès aux liens faibles.

---
*-XX:+UseNUMA* : Active l'entrelacement NUMA sur les hôtes avec plusieurs sockets, lorsqu'il est combiné avec AlwaysPreTouch, il fournit de meilleures performances que la configuration par défaut. Vous trouverez plus de détails sur cette architecture [ici](https://en.wikipedia.org/wiki/Non-uniform_memory_access).

*-XX:+AlwaysPreTouch* : pré-enregistrement de toute la mémoire allouée en une seule fois, réduit les délais d'entrée.

*-XX:-UseBiasedLocking* : Il existe un compromis entre la bande passante du verrouillage illimité (biaisé) et les points sûrs que la JVM réalise pour les activer et les désactiver selon les besoins. Pour les charges de travail axées sur la latence, y compris les serveurs Minecraft, il est judicieux de désactiver le blocage biaisé.

*-XX:+DisableExplicitGC* : L'appel de `System.gc()` à partir d'un code personnalisé oblige ShenandoahGC à effectuer un cycle supplémentaire de garbage collection, la désactivation protège contre les abus de code.

## Logiciel serveur (noyau)

Pour l'option la plus stable et efficace, je recommande [Airplane](https://github.com/TECHNOVE/Airplane).

## Système

Tuned-adm est un outil en ligne de commande qui vous permet de passer d'un profil à l'autre pour améliorer les performances dans un certain nombre de cas d'utilisation spécifiques.  Installez le paquet avec `apt` :

```yml
sudo apt install tuned
```

Ensuite, vous devez choisir la configuration pour votre système, je recommande d'utiliser `throughput-performance` ou `latency-performance`, définissez le profil dont vous avez besoin :

```yml
sudo tuned-adm profile throughput-performance
```

Vous pouvez vérifier que les changements ont été appliqués avec la commande `tuned-adm profile`.

Un article détaillé sur tous les profils et quand les utiliser [ici](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-tool_reference-tuned_adm).

## Configuration supplémentaire

### bukkit.yml

```yml
chunk-gc :
 period-in-ticks : 600
```

**Valeur recommandée pour `chunk-gc.period-in-ticks`:**  
N'allouez pas plus de 12 Go de mémoire, cela n'affectera pas la plupart des cas.
| Mémoire / Nombre de joueurs | jusqu'à 30 | de 30 à 60 | de 60 à 100 | plus de 100 |
| :--- | :---: | :---: | :---: | :---: |
| 4 GB | 400 | - | - | - | |
| 8 GB | 600 | 400 | 300 | - |
| 12 GB | 1200 | 800 | 600 | 400 |
