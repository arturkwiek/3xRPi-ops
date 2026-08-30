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

## Etap 6 — baseline i monitoring, ale **bez rpi-02**

```bash
ansible-playbook playbooks/site.yml --limit rpi-01,rpi-03 --check --diff
ansible-playbook playbooks/site.yml --limit rpi-01,rpi-03
```

**Dlaczego bez rpi-02 — to nie jest ostrożność, tylko fakt o kodzie.** Rola
`node_exporter` **nie instaluje** node_exportera; ona tylko pilnuje, żeby istniejąca
usługa chodziła (`service: started` + `wait_for` portu 9100). Na rpi-01 i rpi-03 ta
usługa jest, na rpi-02 **nie ma ani binarki, ani unitu**, więc bieg zatrzyma się tam
na błędzie.

Po tym etapie sprawdź w Prometheusie, czy oba cele są `up`:
<http://localhost:9090/targets>

---

## Etap 7 — wyrównanie rpi-02

```bash
ansible-playbook playbooks/update.yml --limit rpi-02
```

Ta maszyna ma **131 pakietów do aktualizacji** i nie ma `pip`. `update.yml` robi
`apt update` + `upgrade: safe`, a restart jest opcjonalny:

```bash
ansible-playbook playbooks/update.yml --limit rpi-02 -e reboot_if_required=true
```

Zostaje `node_exporter`. Dopóki rola go nie instaluje, masz dwie drogi: wgrać binarkę
i unit ręcznie (wzorem rpi-01: `/usr/local/bin/node_exporter`, unit `node_exporter`,
wersja upstream 1.11.1) albo poczekać, aż rola dostanie krok `get_url` + szablon unitu.

Dopiero gdy `:9100` na rpi-02 odpowiada, odkomentuj trzeci cel w
`~/3xRPi/config/prometheus.yml` — wcześniej byłby stałym `HostDown`.

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
