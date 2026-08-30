# Ściąga — 3xRPi-ops w jednej kartce

Jedna strona do trzymania otwartej **obok terminala**. Nie tłumaczy Ansible od zera —
od tego jest [`wprowadzenie-ansible.md`](wprowadzenie-ansible.md). To jest to, co się
pamięta źle albo wcale.

**Kolejność czytania, jeśli wracasz po przerwie:** ta kartka → sekcja „Stan na dziś"
na jej końcu → dopiero potem `README.md`, jeśli szukasz szczegółu.

---

## Mapa repo — co jest czym

```
ansible.cfg                     jak się łączyć (pyta o hasła, ignoruje known_hosts)
inventory/
  hosts.ini                     KTO — trzy adresy + historia wszystkich przeadresowań
  group_vars/rpi.yml            zmienne dla całej grupy (m.in. ansible_user: mwd)
  host_vars/rpi-0*.yml          zmienne dla JEDNEJ maszyny (profile obciążenia demo)
playbooks/
  site.yml       CO: baseline + node_exporter          ← konfiguracja stała
  identity.yml   CO: nadaj płytom tożsamość            ← jednorazowy, NIGDY nieuruchomiony
  update.yml     CO: apt update + safe upgrade
  reboot.yml     CO: restart po jednej płycie, tylko gdy system o to prosi
  stress.yml     CO: sztuczne obciążenie pod demo monitoringu
roles/
  baseline/       pakiety, strefa czasowa, unattended-upgrades
  node_exporter/  pilnuje, żeby usługa :9100 chodziła   ← UWAGA: nie instaluje, patrz niżej
  identity/       hostname (MWDRPi-1/-2/-3), klucze hosta, konto admina  ← najzawilsza
  app/            pusty szkielet, celowo niewpięty w site.yml
```

Zależność, którą warto mieć w głowie:

```
inventory (KTO) + playbook (CO)  →  ansible-playbook
                     └── playbook przypisuje ROLE hostom
                           └── rola to lista ZADAŃ
                                 └── zadanie wywołuje MODUŁ (apt, service, command…)
```

---

## Pięć komend, które wystarczą

```bash
cd ~/3xRPi-ops

ansible rpi -m ping                                    # 1. czy w ogóle wchodzę na maszyny
ansible-playbook playbooks/site.yml --check --diff     # 2. co BY zrobił, nic nie zmienia
ansible-playbook playbooks/site.yml                    # 3. to samo na ostro
ansible-playbook playbooks/site.yml --limit rpi-01     # 4. tylko jedna maszyna
ansible-playbook playbooks/update.yml                  # 5. apt update + upgrade
ansible-playbook playbooks/reboot.yml                  # 6. restart tych, ktore o to prosza
```

Każdy bieg zapyta o **dwa** hasła: `SSH password` i `BECOME password`. Oba to hasło
użytkownika `mwd`. Tak jest ustawione w `ansible.cfg` (`ask_pass`, `become_ask_pass`)
i dlatego **żaden bieg nie pójdzie w tle ani z crona** — wymaga człowieka przy klawiaturze.

---

## Jak czytać wynik

```
ok=4    zadanie sprawdzone, stan już był poprawny — nic nie zrobiono
changed=2   zadanie COŚ ZMIENIŁO na maszynie
skipped=1   pominięte (warunek `when` nie zaszedł albo tryb --check)
failed=1    zadanie się wywaliło; dalsze zadania na TYM hoście się nie wykonają
unreachable=1   nie udało się nawet połączyć — zły adres, hasło albo host nie żyje
```

**Drugi bieg tego samego playbooka powinien dać same `ok`, zero `changed`.** Jeśli
`changed` wraca za każdym razem, to znak, że zadanie jest napisane nieidempotentnie —
to najczęstszy błąd początkującego i główny powód, dla którego warto patrzeć na te liczby.

---

## Złota zasada i jej granica

**Zawsze najpierw `--check --diff`.** Ale nie traktuj tego jak wyroczni:

- moduły `command` i `shell` są w trybie `--check` **pomijane**, a nie symulowane —
  a `baseline` używa `command` do strefy czasowej. Dry-run pokaże więc **mniej**,
  niż zrobi bieg na ostro;
- jeśli zadanie A miałoby coś zmienić, a zadanie B zależy od efektu A, to w `--check`
  B zobaczy stary stan i może zgłosić błąd, którego naprawdę nie będzie.

Wniosek: `--check` łapie literówki i grube pomyłki, nie zastępuje myślenia.

---

## Pułapki tej konkretnej floty

| Pułapka | Co się stanie | Co zrobić |
|---|---|---|
| ~~`node_exporter` NIE instaluje~~ — **naprawione 30-08** | rola instaluje tam, gdzie brakuje, i adoptuje tam, gdzie już jest. `site.yml` obejmuje całą trójkę | pierwszy bieg na rpi-02 zrób osobno: `--limit rpi-02 --check --diff`, potem na ostro |
| **`fleet_regenerate_machine_id: true`** w `roles/identity/defaults/main.yml:24` | zmieni DUID DHCP → płyta wróci pod **innym adresem** | pomiar z 30-08 obalił przesłankę tej opcji (wszystkie trzy `machine-id` są różne). Przestaw na `false`, zanim uruchomisz `identity.yml` |
| **Płyty są tylko na Wi-Fi**, `eth0` jest `down` na wszystkich trzech | błąd w konfiguracji sieci = koniec dostępu, maszyny są bezgłowe | `identity` **celowo nie tyka netplana**. Nie dopisuj tam zadań sieciowych |
| **`fleet_disable_password_auth`** | włączone przed wgraniem klucza = zamknięcie się na zewnątrz | zostaw `false`, dopóki nie zalogujesz się kluczem na **każdej** płycie |
| **Adresy z DHCP, rozjechały się 4× w miesiąc** | playbook wisi na `unreachable` albo trafia w cudzą maszynę | `~/3xRPi/scripts/find_pi.sh`, potem popraw `hosts.ini`. Docelowo: rezerwacje po MAC na routerze |
| **`sudo` bez hasła jest tylko na rpi-01 i rpi-03** | na rpi-02 `become` wymaga hasła | `become_ask_pass` jest włączone globalnie właśnie dlatego — nie wyłączaj |
| **`identity.yml` ma `serial: 1`** | leci maszyna po maszynie, nie równolegle | to celowe: po zmianie tożsamości płyta może wrócić pod innym adresem |

---

## Słowniczek — dziesięć słów, które wystarczą

| Słowo | Znaczenie w jednym zdaniu |
|---|---|
| **inventory** | lista maszyn i ich adresów |
| **play** | „zrób TO na TYCH hostach" — jeden blok w playbooku |
| **playbook** | plik z jednym lub kilkoma play |
| **rola** | paczka zadań wielokrotnego użytku (`tasks/`, `defaults/`, `handlers/`) |
| **task** | jedno zadanie; wywołuje moduł |
| **moduł** | `apt`, `service`, `copy`, `command` — cegiełka, która coś robi |
| **handler** | zadanie uruchamiane **tylko gdy** ktoś je „powiadomi" (`notify`), np. restart usługi |
| **idempotencja** | drugi bieg nic nie zmienia, bo stan już jest docelowy |
| **become** | „zrób to jako root" (przez `sudo`) |
| **fact** | informacja zebrana automatycznie z maszyny (system, adresy, pamięć) |

---

## Stan na dziś — 2026-08-30

**Żaden playbook z tego repo nie został nigdy uruchomiony na flocie.** Potwierdzone
pomiarem: znacznik `/etc/fleet-identity` nie istnieje na żadnej z trzech płyt,
`authorized_keys` są puste wszędzie. To repozytorium jest dziś **poprawnie napisanym
opisem zamiaru**, nie stanem maszyn.

Flota przestała być jednorodna:

| Host | Adres | System | node_exporter | sudo |
|---|---|---|---|---|
| rpi-01 | `.170` | Ubuntu 26.04 | działa | NOPASSWD |
| rpi-02 | `.172` | **Ubuntu 24.04** | **brak** | z hasłem |
| rpi-03 | `.100` | Ubuntu 26.04 | działa | NOPASSWD |

Pełna procedura z warunkami przerwania na każdym etapie: **[`SYNCHRONIZACJA.md`](SYNCHRONIZACJA.md)**.
W skrócie:

1. `ansible rpi -m ping` — **przeszedł 30-08 na wszystkich trzech.**
2. `ansible rpi -m command -a 'id -un' --become` — to dopiero sprawdza `sudo`, czego `ping` nie robi.
3. `identity.yml --limit rpi-01 --check --diff`, potem na ostro; po biegu `ssh-keygen -R <adres>`.
4. To samo dla `rpi-03`, na końcu `rpi-02` (nie jest klonem, ale `sudo` wymaga tam hasła).
5. `site.yml` **tylko** na `rpi-01,rpi-03`; rpi-02 potrzebuje wcześniej node_exportera.

`fleet_regenerate_machine_id` jest już przestawione na `false` (commit `370a450`), więc płyta
po biegu `identity.yml` **wraca pod tym samym adresem**.

---

## Dokumenty w tym repo

| Plik | Co daje | Kiedy czytać |
|---|---|---|
| `docs/SCIAGA.md` | ta kartka | zawsze, przy terminalu |
| `docs/SYNCHRONIZACJA.md` | **runbook: doprowadzenie floty do spójnego stanu, etap po etapie** | gdy siadasz do synchronizacji |
| `docs/wprowadzenie-ansible.md` | kurs Ansible od zera na tym projekcie, 12 rozdziałów | raz, na spokojnie |
| `docs/demo-obciazenie.md` | `host_vars` i warunkowe budowanie komend, na przykładzie `stress.yml` | gdy zechcesz zróżnicować maszyny |
| `README.md` | szczegóły techniczne i uzasadnienia decyzji — **po angielsku** | gdy szukasz „dlaczego tak" |
