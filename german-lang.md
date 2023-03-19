> **Hilfe mit der Übersetzung**  
> Falls du Fehler oder Mängel in der Übersetzung findest, bitte informiere @DasSharkk und korrigiere sie. Danke!

## Warum nicht [Aikar flags](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/)?

Sein Müllsammler basiert auf dem G1 Algorithmus. Wie er sagte, ist der Algorithmus unglaublich stabil, aber ist sehr langsam, verglichen zu aktuellen Standards. Gleichzeitig ist es nicht mehr aktuell, da er zum Zeitpunkt von JDK 8 innovativ war, inzwischen aber nicht mehr. Aber warum sollte man etwas ändern, wenn es funktioniert? Naja, du solltest!

Ich ersetze ihn mit Shenandoah - ein Müllsammler mit einer unglaubliche kurzen Pausenzeit, was gut für unser Lieblingsspiel ist, weil wir alle keine Standbilder mögen. Während ich diesen getestet habe, blieb die Stabilität unbeeinträchtigt und ich habe kein einziges Problem gefunden.

## Flaggen

**Unterstützte JDKs:**

*Ich empfehle das Nutzen von OpenJDK 17.*

- [x] OpenJDK 8+
- [x] Red Hat 8+
- [x] Amazon 11+
- [x] Azul 11+
- [x] AdoptOpenJDK 11+
- [ ] Oracle
- [ ] SAP

**Unterstützte Server:**

- [x] Vanilla
- [x] Bukkit, Spigot, Paper ...
- [x] Fabric
- [x] Forge

**Fertige Einstellungen:**

```yml
java -jar -server -Xms6G -Xmx6G -XX:+UseLargePages -XX:LargePageSizeInBytes=2M -XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu -XX:+UseNUMA -XX:+AlwaysPreTouch -XX:-UseBiasedLocking -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 launcher-airplane.jar --nogui
```

> Die Warnung `Option UseBiasedLocking was deprecated in version 15.0 and will likely be removed in a future release.` kann ohne Probleme ignoriert und auf deinem Server verwendet werden, da die UseBiasedLocking Flagge gut funktioniert.

**Und jetzt werden wir jede Einstellung genau analysieren:**

 *-Xms6G* und *-Xmx6G* setzen die Grenzen of der Arbeitsspeicher Nutzung deines Minecraft Servers. Ich empfehle, dass du deinem Server nicht mehr als 12GB zuweist und immer mindestens 1 - 2 GB für das System übrig zu lassen.

 *-XX:+UseLargePages* und *-XX:LargePageSizeInBytes = 2M*: erlaubt, dass große Seiten an Arbeitsspeicher genutzt werden, verbessert Start und Antwotzzeit.  Lass uns Linux Seiten für uns erstellen.  Füge diese Zeile zu `/etc/sysctl.conf` hinzu:

```yml
vm.nr_hugepages = 3372
```

Wie kommen wir auf diese Zahl? Wenn wir 6 GB große Seiten zuweisen wollen, teilen wir 6 GB durch 2.

```yml
6 * 1024 / 2 = 3072
```

Außerdem empfehle ich etwas Platz freizulassen, indem wir 300 zu unserer Nummer addieren.

```yml
3072 + 300 = 3372
```

Dann starten wir das System neu, um die Änderungen zu aktivieren. Mit dem Command `grep -i hugepages /proc/meminfo` kannst du sicherstellen, dass der Arbeitsspeicher richtig registriert wurde.

---
*-XX:+UnlockExperimentalVMOptions*: Aktiviert die Nutzung von noch nicht veröffentlichten Optionen.

*-XX:+UseShenandoahGC*: nutzt das Shenandoah project als Müll-Sammlungs-Algorithmus.

*-XX:ShenandoahGCMode=iu*: Aktiviert den experimentellen Modus des Assemblers, welcher ein Spiegel des SATB Modus ist, durch den das Markup weniger konservativ wird, insbesondere im Hinblick auf den Zugriff auf schwache Links.

---
*-XX:+UseNUMA*: Aktiviert die NUMA Verschachtelung auf Hosts mit mehreren Sockeln, was, wenn man es mit AlwaysPreTouch kombiniert, bessere Performance bietet, als die Standard out-of-box Konfiguration.  Weitere Infos über diese Architektur können [hier](https://en.wikipedia.org/wiki/Non-uniform_memory_access) gefunden werden.

*-XX:+AlwaysPreTouch*: Registriert den gesamten zugewiesenen Arbeitsspeicher auf einmal, reduziert Eingabeverzögerungen.

*-XX:-UseBiasedLocking*: Es gibt einen Kompromiss zwischen der Bandbreite der unbegrenzten (biased) Sperrung und den safe points, die die JVM macht, um sie nach Bedarf ein- und auszuschalten. Für latenzorientierte Workloads, einschließlich Minecraft-Server, ist es sinnvoll, biased blocking zu deaktivieren.

*-XX:+DisableExplicitGC*: Ruft `System.gc()` aus benutzerdefiniertem Code auf, zwingt ShenandoahGC dazu einen zusätzlichen Müllsammlungszyklus auszuführen, das Deaktivieren schützt vor Missbrauch des Codes.

## Server Software (core)

Für die stabliste und effizienteste Variante würde ich [Airplane](https://github.com/TECHNOVE/Airplane) empfehlen.

## System

Tuned-adm ist ein Command-Line Tool, das einem erlaubt zwischen abgestimmten Profilen zu wechseln, um die Performance in vielen speziellen Fällen zu verbessern.  Installiere das Package mit `apt`:

```yml
sudo apt install tuned
```

Als nächstest musst die die Konfiguration für dein System auswählen, ich empfehle `throughput-performance` oder `latency-performance` zu nutzen. Setze das Profil, das du benötigst mit:

```yml
sudo tuned-adm profile throughput-performance
```

Mit dem Command `tuned-adm profile` kannst du sicherstellen, dass die Änderungen übernommen wurden.

Einen detaillierten Artikel über alle Profile und wann du sie nutzen solltest findest du [hier](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-tool_reference-tuned_adm).

## Weitere Konfiguration

### bukkit.yml

```yml
chunk-gc:
 period-in-ticks: 600
```

**Empfohlener Wert für `chunk-gc.period-in-ticks`:**

Weise nicht mehr als 12 GB Arbeitsspeicher zu, weil es die meisten Fälle nicht betreffen wird.

| Arbeitsspeicher / Spieleranzahl | bis 30 | 30 - 60 | 60 - 100 | über 100 |
| :--- | :---: | :---: | :---: | :---: |
| 4 GB | 400 | - | - | - |
| 8 GB | 600 | 400 | 300 | - |
| 12 GB | 1200 | 800 | 600 | 400 |
