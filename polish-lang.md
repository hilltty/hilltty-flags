> **Pomoc z przetłumaczeniem**
> Jeśli zobaczysz jakieś problemy, albo niedociągnięcia w przetłumaczeniu, proszę, zgłoś je i popraw. Dziękuję.
## Czemu nie [Aikar flags](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/)?
To jest bardzo proste. Jego flagi używają algorytmu GC: G1. Jak powiedział, ten algorytm jest bardzo stabilny, ale jest ekstremalnie wolny według teraźniejszych standardów. I też jest bardzo przestarzały, wszystko to co to, zaimplementowało, było nowoczesne w dniach JDK 8, ale teraz już nie. Racja, czemu zmieniać coś, co działa? Cóż, powinieneś.

Moją propozycją, jest to, żeby zamienić to na Shenandoah - ten algorytm GC ma bardzo ekstremalnie krótki czas pauzy, który bardzo pasuje do naszej ulubionej gry, wszyscy nie lubimy zacinania. To nie zmienia stabilności w żaden sposób, podczas całego czasu nie, przerwanego testowania, nie zidentyfikowałem żadnego problemu.
## Odmowa odpowiedzialności
Nie namawiam wszystkich do natychmiastowej zmiany właściwości uruchamiania serwera, chcę tylko wyjaśnić, że nic nie jest idealne. Ponadto nie jestem odpowiedzialny za stabilność moich parametrów w danym przypadku użycia, wszystkie systemy są różne, a wyniki mogą się różnić.
## Flagi
**Obsługiwane rodzaje/wersje JDK:**

*Rekomenduję użycie OpenJDK 16*
- [x] OpenJDK 8+
- [x] Red Hat 8+
- [x] Amazon 11+
- [x] Azul 11+
- [x] AdoptOpenJDK 11+
- [ ] Oracle
- [ ] SAP

**Obsługiwane serwery:**
- [x] Vanilla
- [x] Bukkit, Spigot, Paper ...
- [x] Fabric
- [x] Forge

**Skończone parametry:**
'''yml
java -jar -server -Xms6G -Xmx6G -XX:+UseLargePages -XX:LargePageSizeInBytes=2M -XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu -XX:+UseNUMA -XX:+AlwaysPreTouch -XX:-UseBiasedLocking -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 launcher-airplane.jar --nogui
'''
> Ostrzeżenie 'Option UseBiasedLocking was deprecated in version 15.0 and will likely be removed in a future release.' może być bezpiecznie zignorowane i użyte na twoim serwerze, flaga 'UseBiasedLocking' wykonuje swoją pracę dobrze.

**A teraz dokładnie przeanalizujemy, co jest odpowiedzialne za co:**

*-Xms6G* and *-Xmx6G*: ustawia limit użycia pamięci przez twój serwer Minecraft, Ja rekomenduje, żeby nie używac więcej niż 12 GB dla twojego serwera i żeby zawsze zostawic 1 - 2 GB wolnej pamięci dla twojego systemu.

*-XX:+UseLargePages* i *-XX:LargePageSizeInBytes = 2M*: **tylko dla zaawansowanych użytkowników**, pozwala użycie dużych stron zarejestowanej pamięci, żeby była użyta, akceleruje prędkość uruchomienia serwera i jego responsywność. Pozwólmy sobie na rejestrację stron w systemie Linux. Dodaj tą linię do '/etc/sysctl.conf':
'''yml
vm.nr_hugepages = 3372
'''
Jak uzyskaliśmy ten numer?, Powiedzmy, że chce zarejestrować 6 GB pamięci dla dużych stron, do tego, dzielę 6 GB przez 2.
'''yml
6 * 1024 / 2 = 3072
'''
Następnie, rekomenduje zostawienie trochę wolnego miejsca i dodania 300 do naszego numeru.
'''yml
3072 + 300 = 3372
'''
Po tym, uruchamiamy ponownie system, żeby zastosować zmiany. Możesz zweryfikować to, że pamięć została pomyślnie zarejestowana z tą komendą 'grep -i hugepages /proc/meminfo'.

---
*-XX:+UnlockExperimentalVMOptions*: pozwala użycie eksperymentalnych funkcji.

*-XX:+UseShenandoahGC*: używa projektu Shenandoah, jako algorytmu GC (tak, tłumacz odczytuje tę nazwę).

*-XX:ShenandoahGCMode=iu*: włączyć tryb eksperymentalny naszego asemblera, jest to lustro trybu SATB, co sprawi, że znaczniki będą mniej konserwatywne, zwłaszcza w odniesieniu do dostępu do słabych ogniw.
---
*-XX:+UseNUMA*: Umożliwia interleaving NUMA na hostach z wieloma gniazdami, w połączeniu z AlwaysPreTouch zapewnia lepszą wydajność niż domyślna konfiguracja out-of-box. Więcej informacji na temat tej architektury można znaleźć [tutaj](https://en.wikipedia.org/wiki/Non-uniform_memory_access).

*-XX:+AlwaysPreTouch*: wstępna rejestracja całej przydzielonej pamięci na raz, zmniejsza opóźnienia wejściowe.

*-XX:-UseBiasedLocking*: Istnieje kompromis między przepustowością nieograniczonego (tendencyjnego) blokowania a bezpiecznymi punktami, które JVM sprawia, aby włączać i wyłączać je w razie potrzeby. W przypadku obciążeń skoncentrowanych na opóźnieniach, w tym serwerów Minecraft, warto wyłączyć blokowanie stronnicze.

*-XX:+DisableExplicitGC*: Uruchamianie System.gc () z kodu niestandardowego wymusza ShenandoahGC do wykonywania dodatkowego cyklu wyrzucania elementów bezużytecznych, wyłączenie chroni przed nadużywaniem kodu.

## Oprogramowanie serwera (rdzeń)
Dla najbardziej stabilnej i wydajnej opcji, polecam [Airplane](https://github.com/TECHNOVE/Airplane).
## System
Tuned-adm to jest narzędzie w terminalu, który pozwala przełączać się między dostrojonymi profilami, aby poprawić wydajność w wielu przypadkach użycia. Zainstaluj pakiet, używając 'apt-get':
'''yml
sudo apt-get install tuned
'''
Następnie musisz wybrać konfigurację dla swojego systemu, polecam użycie profilu 'throughput-performance', albo ' latency-performance', którego potrzebujesz:
'''yml
sudo tuned-adm profile throughput-performance
'''
Można sprawdzić, czy zmiany zostały zastosowane za pomocą polecenia 'tuned-adm profile'.

Szczegółowy artykuł o wszystkich profilach i o tym, kiedy z nich korzystać [tutaj](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-tool_reference-tuned_adm).
## Dodatkowa konfiguracja
### bukkit.yml
'''yml
chunk-gc:
period-in-ticks: 600
'''
**Rekomendowana wartośc dla 'chunk-gc.period-in-ticks':**
Nie należy przydzielić więcej niż 12 GB pamięci, nie będzie to miało wpływu na większość przypadków.
| Pamięć / Liczba gracz | do 30 | 30 - 60 | 60 - 100 | ponad 100 |
| :--- | :---: | :---: | :---: | :---: |
| 4 GB | 400 | - | - | - |
| 8 GB | 600 | 400 | 300 | - |
| 12 GB | 1200 | 800 | 600 | 400 |