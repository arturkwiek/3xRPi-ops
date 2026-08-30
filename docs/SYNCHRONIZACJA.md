# Synchronizacja floty — runbook krok po kroku

Do wykonania **przez Ciebie, przy terminalu**. Każdy bieg pyta o hasła, więc nic z tego
nie da się uruchomić w tle ani zdalnie.

**Co tu znaczy „synchronizacja":** doprowadzić trzy płyty do stanu, w którym mają własne
nazwy, logujesz się kluczem zamiast hasłem, mają ten sam zestaw pakietów bazowych
i wszystkie trzy wystawiają metryki na `:9100`. Dziś nie mają nic z tych czterech rzeczy.

**Zasada nadrzędna:** po każdym etapie jest **warunek przerwania**. Jeśli nie jest
spełniony, nie idź dalej — na maszynach bez monitora, dostępnych tylko po Wi-Fi, cofanie
się jest drogie.

---

## Etap 0 — sprawdź grunt (2 min)

```bash
~/3xRPi/scripts/find_pi.sh                # czy adresy z inwentarza jeszcze żyją
cd ~/3xRPi-ops
ansible rpi -m ping
```

Adresy w tej sieci rozjechały się **cztery razy w miesiąc**. Jeśli `find_pi.sh` pokaże
inne niż `.170` / `.172` / `.100`, popraw `inventory/hosts.ini` **zanim** cokolwiek
uruchomisz — inaczej trafisz w cudzą maszynę albo playbook zawiśnie na `unreachable`.

**Warunek przerwania:** trzy razy `SUCCESS` / `pong`. Mniej — napraw adresy i powtórz.

---

## Etap 1 — sprawdź `sudo`, bo `ping` tego nie sprawdził (1 min)

```bash
ansible rpi -m command -a 'id -un' --become
```

Moduł `ping` **nie podnosi uprawnień**, więc wczorajszy zielony wynik nic nie mówi
o `sudo`. A `sudo` jest tu niesymetryczne: rpi-01 i rpi-03 mają `NOPASSWD`, **rpi-02
wymaga hasła**. Ta komenda jest pierwszym miejscem, w którym to wyjdzie.

**Oczekiwane:** `root` z wszystkich trzech.
**Warunek przerwania:** jeśli rpi-02 zwróci `Missing sudo password` albo
`sudo: a password is required` — hasło BECOME nie przeszło. Bez tego `identity.yml`
i `site.yml` nie ruszą na tej maszynie.

---

## Etap 2 — `identity` na sucho, jedna maszyna (3 min)

```bash
ansible-playbook playbooks/identity.yml --limit rpi-01 --check --diff
```

Czytasz to jak listę zamiarów. Powinieneś zobaczyć zamiar utworzenia `~/.ssh`,
wstawienia klucza publicznego do `authorized_keys`, ustawienia nazwy hosta na `rpi-01`
i dopisania `preserve_hostname: true` do cloud-inita.

**Nie zdziw się, że lista jest krótsza, niż się spodziewasz.** W trybie `--check`
moduły `command` są **pomijane, nie symulowane** — a ustawienie nazwy hosta idzie
właśnie przez `command`. Dry-run pokaże mniej, niż zrobi bieg na ostro.

**Warunek przerwania:** jakikolwiek `failed`. Zwłaszcza komunikat o braku klucza
publicznego — rola sprawdza to celowo na samym początku, żebyś dowiedział się o tym
przed, a nie w połowie roboty.

---

## Etap 3 — `identity` na ostro, tylko rpi-01 (5 min)

```bash
ansible-playbook playbooks/identity.yml --limit rpi-01
```

**Co się wydarzy, po kolei:**

1. Konto `mwd` dostaje Twój klucz publiczny (`~/.ssh/id_ed25519_rpi.pub`).
2. `sshd_config` dostaje `PubkeyAuthentication yes`. Plik jest **walidowany przed
   zapisem** (`sshd -t`), więc literówka nie zamknie Ci dostępu.
3. Nazwa hosta zmienia się z `MWDRPi` na `rpi-01`.
4. Klucze hosta SSH są generowane od nowa.
5. Handler `reboot board` restartuje płytę, a Ansible czeka na jej powrót (do 300 s).

**Adres NIE powinien się zmienić.** Regeneracja `machine-id` jest wyłączona od
commita `370a450`, więc DUID DHCP zostaje ten sam i płyta wraca pod `.170`. Gdyby była
włączona, wróciłaby gdzie indziej — dlatego jest wyłączona.

**Po biegu, dwie rzeczy do zrobienia ręcznie:**

```bash
# klucze hosta się zmieniły, a Twój known_hosts jest HASHOWANY (linie |1|...),
# więc grepem tego nie znajdziesz — użyj -R:
ssh-keygen -R 192.168.0.170

# test właściwy: logowanie KLUCZEM, bez pytania o hasło
ssh -i ~/.ssh/id_ed25519_rpi mwd@192.168.0.170 hostnamectl --static
```

**Warunek przerwania:** jeśli druga komenda pyta o hasło albo nie zwraca `rpi-01`,
zatrzymaj się. Nie rób etapu 3 na kolejnych płytach, dopóki na pierwszej nie działa —
bo wtedy popełniłbyś ten sam błąd trzy razy.

> Ansible ma `host_key_checking = False`, więc **jemu** zmiana klucza hosta nie
> przeszkadza. Krzyczeć będzie Twój ręczny `ssh` — i to jest poprawne zachowanie,
> nie usterka.

---

## Etap 4 — powtórz na rpi-03, potem rpi-02

```bash
ansible-playbook playbooks/identity.yml --limit rpi-03
ansible-playbook playbooks/identity.yml --limit rpi-02
```

Za każdym razem: `ssh-keygen -R <adres>` i test logowania kluczem.

**rpi-02 jest inna i warto wiedzieć, w czym.** Nie jest klonem — stoi na świeżym
Ubuntu 24.04 z lutego, ma własne klucze hosta i własną nazwę do ustawienia, ale
**`sudo` wymaga tam hasła**. Jeśli któryś etap ma się wyłożyć, to ten.

---

## Etap 5 — przestaw się na klucz (2 min, i bardzo się opłaca)

Gdy wszystkie trzy logują się kluczem, w `ansible.cfg`:

```ini
[defaults]
ask_pass         = False
private_key_file = ~/.ssh/id_ed25519_rpi
```

Od tej chwili biegi nie pytają o hasło SSH. **`become_ask_pass` zostaw włączone** —
rpi-02 nadal potrzebuje hasła do `sudo`.

> Nie włączaj `fleet_disable_password_auth`. To zamyka logowanie hasłem na maszynach
> bez monitora — sensowne dopiero, gdy klucz działa na **każdej** płycie i masz pewność,
> że go nie zgubisz.

---

## Etap 6 — baseline i monitoring

```bash
ansible-playbook playbooks/site.yml --limit rpi-01,rpi-03 --check --diff
ansible-playbook playbooks/site.yml --limit rpi-01,rpi-03
```

Na tych dwóch płytach rola `node_exporter` działa w trybie **adopcji** — usługa już
tam jest, więc rola jej nie dotyka i pilnuje tylko, żeby chodziła. Poprawny wynik to
same `ok`, zero `changed`.

**rpi-02 zrób osobno i najpierw na sucho**, bo to jedyna maszyna, na której rola
faktycznie coś zainstaluje:

```bash
ansible-playbook playbooks/site.yml --limit rpi-02 --check --diff
ansible-playbook playbooks/site.yml --limit rpi-02
```

Do 30-08 rola **tylko adoptowała** i bieg na rpi-02 kończył się tam błędem. Od
commita, który dołożył krok instalacyjny, rola pobiera binarkę z upstreamu
(zweryfikowaną sumą z `sha256sums.txt` wydania), zakłada konto systemowe
`node_exporter` i zapisuje jednostkę systemd — przepisane z oryginalnego
`setup_rpi_monitoring.sh`, żeby ta płyta była nieodróżnialna od pozostałych dwóch.
Pierwsze zadanie roli wypisuje, w którym trybie działa, więc od razu widać, czy
adoptuje, czy instaluje.

Po tym etapie sprawdź w Prometheusie, czy oba cele są `up`:
<http://localhost:9090/targets>

---

## Etap 7 — wyrównanie rpi-02

```bash
ansible-playbook playbooks/update.yml --limit rpi-02
```

> **Sprawdź to najpierw — stan mógł się zmienić bez Ciebie.** Pomiar 131 pakietów
> pochodzi z 30-08 00:07, a `unattended-upgrades` jest aktywne na wszystkich trzech
> płytach. Tego samego dnia po południu rpi-02 meldowała już `0 updates can be applied
> immediately` **i `*** System restart required ***`** — czyli najpewniej połatała się
> sama i czeka wyłącznie na restart. Jedna komenda rozstrzyga:
>
> ```bash
> ssh mwd@192.168.0.172 'apt list --upgradable 2>/dev/null | grep -c upgradable; uptime -s'
> ```
>
> Jeśli wyjdzie `0`, pomiń `update.yml` i zrób sam restart:
>
> ```bash
> ansible-playbook playbooks/reboot.yml --limit rpi-02
> ```

Jeśli aktualizacje faktycznie czekają: `update.yml` robi `apt update` + `upgrade: safe`,
a restart jest opcjonalny:

```bash
ansible-playbook playbooks/update.yml --limit rpi-02 -e reboot_if_required=true
```

`node_exporter` załatwia już `site.yml` z etapu 6 — rola instaluje go tam, gdzie
brakuje. Ręczne wgrywanie binarki nie jest potrzebne.

Dopiero gdy `:9100` na rpi-02 odpowiada, odkomentuj trzeci cel w
`~/3xRPi/config/prometheus.yml` — wcześniej byłby stałym `HostDown`.

Wtedy zniknie też ostatni objaw tej samej luki: `find_pi.sh` przestanie zgłaszać
„nazwa hosta nieznana" dla `.172`, bo czyta ją właśnie z metryk na `:9100`.

---

## Co zrobić, gdy…

| Objaw | Przyczyna | Co zrobić |
|---|---|---|
| `unreachable` na którymś hoście | adres z DHCP się zmienił albo płyta wyłączona | `~/3xRPi/scripts/find_pi.sh`, popraw `hosts.ini` |
| `Missing sudo password` | hasło BECOME nie przeszło (najpewniej rpi-02) | powtórz bieg; sprawdź hasło komendą z etapu 1 |
| `Permission denied (publickey,password)` | zły użytkownik | logujesz się jako `mwd`, nie `rpi-01` — to alias z inwentarza, nie konto |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` | klucze hosta zmienione przez `identity` — **spodziewane** | `ssh-keygen -R <adres>` |
| Płyta nie wraca po restarcie | handler `reboot board` czeka 300 s | poczekaj, potem `find_pi.sh`; sprawdź, czy nie wróciła pod innym adresem |
| Bieg pokazuje `changed` przy **każdym** uruchomieniu | zadanie nie jest idempotentne | to błąd w zadaniu, nie w Twoim biegu — zgłoś, przyjrzę się |
| `--check` przechodzi, a bieg na ostro się wywala | `command`/`shell` są w `--check` pomijane | normalne; `--check` łapie literówki, nie zastępuje myślenia |

---

## Jedna rzecz, której ten runbook nie rozwiąże

Adresy będą się rozjeżdżać dalej, bo przyczyna czterech dryfów **pozostaje
nierozpoznana** — hipoteza `machine-id` została obalona pomiarem 30-08. Jedynym
znanym rozwiązaniem są **rezerwacje DHCP po MAC na routerze**:

```
rpi-01  88:a2:9e:27:38:7a
rpi-02  88:a2:9e:27:39:49
rpi-03  88:a2:9e:27:2c:be
```

To są MAC-i interfejsów `wlan0` — `eth0` jest na wszystkich trzech płytach wyłączony.
Bez tych rezerwacji każdy plik z adresami, łącznie z `hosts.ini` i `prometheus.yml`,
ma datę ważności liczoną w dniach.
