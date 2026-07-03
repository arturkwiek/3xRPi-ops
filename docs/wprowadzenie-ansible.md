# Wprowadzenie do Ansible — na przykładzie tego projektu

Ten dokument tłumaczy Ansible od zera, ale zamiast abstrakcyjnych przykładów
używa **plików, które masz w tym repozytorium**. Po przeczytaniu powinieneś
rozumieć, co robi każdy plik w `3xRPi-ops` i dlaczego.

---

## 1. Czym w ogóle jest Ansible?

Wyobraź sobie, że masz 3 Raspberry Pi i chcesz na każdym z nich:
zainstalować te same pakiety, ustawić tę samą strefę czasową, uruchomić
monitoring. Możesz zalogować się przez SSH na każdego po kolei i wpisywać
komendy ręcznie — ale to:

- **nudne** (3× to samo),
- **podatne na błędy** (na rpi-02 zapomniałeś jednej komendy),
- **nie do odtworzenia** (za pół roku nie pamiętasz, co dokładnie zrobiłeś).

**Ansible robi to za Ciebie.** Opisujesz *stan docelowy* ("chcę, żeby vim był
zainstalowany, strefa czasowa to Europe/Warsaw...") w plikach tekstowych, a
Ansible loguje się po SSH na wszystkie maszyny i doprowadza je do tego stanu.

Trzy kluczowe cechy:

1. **Agentless** — na Raspberry Pi *nie* instalujesz żadnego dodatkowego
   oprogramowania. Ansible potrzebuje tylko SSH + Pythona (który i tak już tam
   jest). Cała "inteligencja" siedzi na Twoim laptopie (WSL).
2. **Deklaratywny** — nie piszesz "wykonaj `apt install vim`", tylko "vim ma
   być zainstalowany". Różnica jest ważna → patrz idempotencja niżej.
3. **Idempotentny** — możesz uruchomić to samo 100 razy pod rząd. Jeśli stan
   już się zgadza, Ansible **nic nie robi**. Uruchamiasz i widzisz, że wszystko
   jest OK, zamiast bać się, że coś zepsujesz powtórnym uruchomieniem.

> **Kontroler i węzły (nodes)**
> - **Kontroler** = maszyna, z której uruchamiasz Ansible = Twój laptop (WSL2 Ubuntu).
> - **Węzły / hosty zarządzane** = 3 Raspberry Pi.
> Ansible instalujesz **tylko na kontrolerze.**

---

## 2. Idempotencja — najważniejsze pojęcie na start

To słowo brzmi groźnie, ale idea jest prosta. Spójrz na wynik, który już
widziałeś:

```
rpi-01 | SUCCESS => { "changed": false, "ping": "pong" }
```

Zwróć uwagę na **`"changed": false`**. Ansible przy każdym zadaniu mówi Ci,
czy *coś zmienił*:

- **`changed: false`** (zwykle na zielono/na czarno) — sprawdziłem, stan już
  się zgadzał, nic nie ruszałem.
- **`changed: true`** (na żółto/pomarańczowo) — musiałem coś zmienić, żeby
  doprowadzić do stanu docelowego (np. faktycznie zainstalowałem pakiet).
- **`FAILED`/`unreachable`** (na czerwono) — błąd.

Dzięki temu drugie uruchomienie tego samego playbooka na zdrowym systemie
powinno dać same `changed: false`. To jest Twój sygnał "nic się nie
rozjechało". To fundamentalnie inne podejście niż skrypt bash, który za każdym
razem ślepo wykonuje `apt install`.

---

## 3. Anatomia Twojego projektu

Oto co masz i jak elementy się łączą. Przejdziemy przez to od góry do dołu.

```
ansible.cfg                ← globalne ustawienia (jak Ansible ma się łączyć)
inventory/
  hosts.ini                ← LISTA maszyn (kto?)
  group_vars/rpi.yml       ← zmienne dla grupy hostów (szczegóły połączenia)
playbooks/
  site.yml                 ← CO zrobić: baseline + monitoring
  update.yml               ← CO zrobić: aktualizacje systemu
roles/
  baseline/                ← wielokrotnego użytku "paczka" zadań: pakiety, TZ...
  node_exporter/           ← "paczka" zadań: monitoring na :9100
  app/                     ← na razie pusty szkielet
```

Zależność mentalna, którą warto zapamiętać:

```
inventory (KTO)  +  playbook (CO)  →  ansible-playbook to uruchamia
                        │
                        └── playbook przypisuje ROLE do hostów
                              └── rola to zestaw ZADAŃ (tasks)
                                    └── zadanie używa MODUŁU (np. apt, service)
```

---

## 4. Inventory — "kto?" (`inventory/hosts.ini`)

To jest lista maszyn, którymi zarządzasz:

```ini
[rpi]
rpi-01 ansible_host=192.168.0.212
rpi-02 ansible_host=192.168.0.213
rpi-03 ansible_host=192.168.0.145
```

- `[rpi]` to **grupa**. Dzięki niej możesz powiedzieć "zrób coś na całej grupie
  rpi" zamiast wymieniać każdy host z osobna. To właśnie ta grupa, której użyłeś
  w komendzie `ansible rpi -m ping`.
- `rpi-01` to **przyjazna nazwa** (alias). Nie musi być prawdziwym hostname.
- `ansible_host=192.168.0.212` mówi Ansible, pod jaki adres IP faktycznie się
  łączyć.

> **Ćwiczenie do zrozumienia:** gdybyś dopisał drugą grupę, np. `[sensory]` z
> jednym Pi, mógłbyś celować w nią osobno: `ansible sensory -m ping`.

---

## 5. Zmienne grupy (`inventory/group_vars/rpi.yml`)

Plik nazywa się `rpi.yml`, bo dotyczy grupy `rpi` — Ansible łączy je po nazwie
automatycznie. Wszystko, co tu wpiszesz, dotyczy **każdego** hosta w tej grupie:

```yaml
ansible_user: mwd                          # na Pi logujemy się jako user "mwd"
ansible_python_interpreter: /usr/bin/python3
```

Zamiast powtarzać `ansible_user=mwd` przy każdym hoście w `hosts.ini`,
zapisujesz to raz tutaj. **DRY** (Don't Repeat Yourself) w praktyce.

Komentarz w tym pliku wspomina, że jak wdrożysz klucze SSH, to tutaj dopiszesz
`ansible_ssh_private_key_file` — i to jest właściwe miejsce na takie rzeczy.

---

## 6. Konfiguracja (`ansible.cfg`)

Ten plik ustawia domyślne zachowanie, żebyś nie musiał podawać flag przy każdej
komendzie. Twój wygląda tak (z komentarzami po co co jest):

```ini
[defaults]
inventory          = inventory/hosts.ini   # gdzie jest lista hostów
roles_path         = roles                  # gdzie szukać ról
host_key_checking  = False                  # nie pytaj o klucz hosta przy 1. połączeniu
ask_pass           = True                   # zawsze pytaj o hasło SSH

[privilege_escalation]
become             = False                  # domyślnie NIE podnoś uprawnień...
become_method      = sudo                   # ...a jak już, to przez sudo
become_ask_pass    = True                   # i pytaj o hasło sudo
```

To właśnie **z tego pliku** biorą się dwa pytania, które widziałeś:

```
SSH password:                          ← bo ask_pass = True
BECOME password[defaults to SSH password]:   ← bo become_ask_pass = True
```

**"Become"** to Ansiblowe słowo na "stań się kimś innym" — w praktyce
`sudo`. Instalacja pakietu wymaga roota, więc zadanie musi się "stać rootem".
Ponieważ sudo na Twoich Pi wymaga hasła (nie ma NOPASSWD), Ansible musi je od
Ciebie pobrać — stąd drugie pytanie.

> Zauważ: `become = False` globalnie, ale w playbookach zobaczysz
> `become: true`. Czyli: domyślnie nie podnosimy uprawnień, ale konkretne plays,
> które tego potrzebują, włączają to u siebie. Bezpieczna domyślna wartość.

---

## 7. Playbook — "co zrobić?" (`playbooks/site.yml`)

Playbook to serce Ansible: plik YAML mówiący, **co** ma się wydarzyć i **na
których** hostach. Twój `site.yml` jest wzorcowo prosty:

```yaml
- name: Baseline + monitoring for the RPi fleet
  hosts: rpi              # na kim? → grupa rpi z inventory
  become: true            # tak, potrzebujemy roota (sudo)
  roles:                  # co uruchomić? → dwie role, po kolei
    - baseline
    - node_exporter
```

Czyta się to niemal jak zdanie po angielsku: "Na hostach `rpi`, z uprawnieniami
roota, zastosuj rolę `baseline`, a potem `node_exporter`."

- **`- name:`** — opis (pojawia się w outpucie, żebyś wiedział, co leci).
- **`hosts: rpi`** — łączy playbook z grupą z inventory. To jest ten "klej".
- **`roles:`** — lista ról do wykonania, **w kolejności**. Najpierw baseline,
  potem monitoring.

Uruchamiasz to komendą:

```bash
ansible-playbook playbooks/site.yml
```

---

## 8. Role — "paczki" zadań wielokrotnego użytku

Rola to sposób na pogrupowanie powiązanych zadań w katalog o ustalonej
strukturze. Zamiast wrzucać 20 zadań do jednego playbooka, dzielisz je na
tematyczne role. Ansible zna tę strukturę katalogów "z automatu":

```
roles/baseline/
  tasks/main.yml       ← lista zadań do wykonania (obowiązkowe)
  defaults/main.yml    ← domyślne wartości zmiennych (łatwe do nadpisania)
```

### 8a. `roles/baseline/tasks/main.yml` — zadania

To jest lista rzeczy do zrobienia na każdym Pi. Rozbierzmy pierwsze zadanie:

```yaml
- name: Install baseline packages
  ansible.builtin.apt:              # ← MODUŁ: obsługa apt
    name: "{{ baseline_packages }}" # ← argument: co zainstalować (zmienna!)
    state: present                  # ← stan docelowy: "ma być zainstalowane"
    update_cache: true
    cache_valid_time: 3600
```

Trzy rzeczy do zauważenia:

1. **Moduł** (`ansible.builtin.apt`) to "wtyczka" wiedząca, jak rozmawiać z
   `apt`. Ansible ma setki modułów (`apt`, `service`, `copy`, `user`,
   `reboot`...). Ty tylko mówisz *co chcesz*, moduł wie *jak to zrobić* i jak
   sprawdzić, czy już jest zrobione (idempotencja!).
2. **`state: present`** — to jest deklaratywność. Nie mówisz "zainstaluj",
   mówisz "ma być obecne". Jak już jest → `changed: false`.
3. **`{{ baseline_packages }}`** — wąsy `{{ }}` to **wstawienie zmiennej**
   (to silnik szablonów Jinja2). Wartość tej zmiennej jest zdefiniowana w
   `defaults/main.yml` — patrz niżej.

Kolejne zadania w tym pliku pokazują ważne wzorce:

```yaml
- name: Read current timezone
  ansible.builtin.command: timedatectl show -p Timezone --value
  register: baseline_current_tz     # ← ZAPISZ wynik do zmiennej
  changed_when: false               # ← to zadanie tylko czyta, nigdy nie "zmienia"

- name: Set timezone
  ansible.builtin.command: "timedatectl set-timezone {{ baseline_timezone }}"
  when: baseline_current_tz.stdout != baseline_timezone   # ← wykonaj TYLKO jeśli trzeba
```

- **`register:`** zapisuje wynik komendy do zmiennej, żeby użyć go dalej.
- **`when:`** to warunek. Tutaj: zmieniaj strefę czasową *tylko* jeśli obecna
  jest inna niż docelowa. To sposób na ręczne zapewnienie idempotencji przy
  module `command` (który sam z siebie nie wie, czy coś zmienił — dlatego
  dodano `changed_when: false`).

To ładnie pokazuje różnicę: moduł `apt` ogarnia idempotencję sam; przy surowym
`command` musisz zadbać o nią ręcznie przez `when` i `changed_when`.

### 8b. `roles/baseline/defaults/main.yml` — zmienne

```yaml
baseline_timezone: Europe/Warsaw

baseline_packages:
  - vim
  - htop
  - curl
  - ca-certificates
  - unattended-upgrades
  - update-notifier-common

baseline_enable_unattended_upgrades: true
```

To są **domyślne wartości** zmiennych używanych w zadaniach. Trzymanie ich
osobno od logiki (zadań) ma ogromną zaletę: żeby zmienić listę pakietów albo
strefę czasową, edytujesz dane, a nie kod. `defaults` mają najniższy priorytet,
więc łatwo je nadpisać (np. w `group_vars` albo flagą `-e` w linii komend).

### 8c. `roles/node_exporter` — druga rola

Analogiczna struktura. Instaluje exporter metryk (Prometheus node_exporter) i
pilnuje, żeby usługa działała na porcie 9100:

```yaml
- name: Ensure node_exporter service is enabled and running
  ansible.builtin.service:
    name: "{{ node_exporter_service }}"
    state: started        # ma działać teraz
    enabled: true         # ma się uruchamiać po restarcie
```

Moduł `service` zarządza usługami systemd. `state: started` + `enabled: true`
to najczęstsza para: "uruchom teraz i włącz autostart".

> **Uwaga z README, którą warto znać:** na Twoich Pi jest już *jakiś*
> node_exporter z ręcznej instalacji na porcie 9100. Jeśli on trzyma port,
> pakietowa usługa się nie podniesie. Dlatego przy tej roli **najpierw robisz
> dry-run** (patrz sekcja 10).

### 8d. `roles/app` — pusty szkielet

Celowo pusta rola-placeholder. Nie jest podpięta do `site.yml`. Czeka, aż
zdecydujesz, co Pi mają hostować. Dobry przykład, że rolę można mieć "w
zanadrzu" bez uruchamiania.

---

## 9. Drugi playbook (`playbooks/update.yml`) — zadania inline

Ten playbook pokazuje alternatywę dla ról: zadania wpisane **wprost w
playbooku** (bo to jednorazowa, prosta operacja, nie warto robić roli):

```yaml
- name: Update & upgrade the RPi fleet
  hosts: rpi
  become: true
  vars:
    reboot_if_required: false     # domyślnie NIE restartuj
  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Safe-upgrade installed packages
      ansible.builtin.apt:
        upgrade: safe

    - name: Check whether a reboot is required
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required_file

    - name: Reboot if required (opt-in)
      ansible.builtin.reboot:
        msg: "Reboot triggered by Ansible update.yml"
      when:
        - reboot_if_required | bool                # tylko jeśli TY się zgodziłeś
        - reboot_required_file.stat.exists         # ...i system faktycznie prosi
```

Świetny przykład bezpiecznych domyślnych: restart wykona się **tylko** gdy
spełnione są *oba* warunki — Ty jawnie włączyłeś `reboot_if_required=true`
**oraz** system utworzył plik `/var/run/reboot-required`. Domyślnie playbook
niczego nie zrestartuje.

Uruchomienie z opcjonalnym restartem:

```bash
ansible-playbook playbooks/update.yml -e reboot_if_required=true
```

Flaga **`-e`** ("extra vars") nadpisuje zmienną z linii komend — ma
najwyższy priorytet, bije wartość z `vars:`.

---

## 10. Twój codzienny workflow

```bash
cd ~/3xRPi-ops

# 1. Sprawdź łączność (to już robiłeś)
ansible rpi -m ping

# 2. ZAWSZE najpierw dry-run: pokaż co BY zrobił, ale nic nie zmieniaj
ansible-playbook playbooks/site.yml --check --diff

# 3. Jak wynik dry-runa wygląda sensownie — zastosuj naprawdę
ansible-playbook playbooks/site.yml

# 4. Aktualizacje systemu
ansible-playbook playbooks/update.yml
```

**`--check`** to tryb próbny ("co by się stało"), a **`--diff`** pokazuje
konkretne różnice w plikach. To Twoja siatka bezpieczeństwa — zwłaszcza przy
roli `node_exporter` (patrz uwaga o porcie 9100 w README).

### Przydatne komendy ad-hoc

Poza playbookami możesz odpalać pojedyncze moduły "na już":

```bash
ansible rpi -m ping                          # test łączności
ansible rpi -m setup                          # zrzuć WSZYSTKIE fakty o hostach
ansible rpi -a "uptime"                        # uruchom dowolną komendę (moduł command)
ansible rpi -m apt -a "name=tree state=present" --become   # zainstaluj pakiet ad-hoc
```

- **`-m`** = wybierz moduł, **`-a`** = argumenty do niego.
- `ansible rpi -a "uptime"` używa domyślnego modułu `command`.

---

## 11. Pojęcia, które teraz rozpoznasz

| Pojęcie | Co to | Gdzie w Twoim repo |
|---|---|---|
| **Inventory** | lista hostów | `inventory/hosts.ini` |
| **Grupa** | zbiór hostów pod jedną nazwą | `[rpi]` |
| **Playbook** | plik YAML: co zrobić i na kim | `playbooks/*.yml` |
| **Play** | jeden blok `- name/hosts/...` w playbooku | każdy `- name:` na górze |
| **Task (zadanie)** | pojedynczy krok | wpisy pod `tasks:` |
| **Module (moduł)** | "wtyczka" robiąca robotę | `ansible.builtin.apt`, `.service`... |
| **Role (rola)** | katalog pogrupowanych zadań | `roles/baseline/` |
| **Fakty (facts)** | dane zebrane o hoście | `ansible rpi -m setup` |
| **Zmienne** | wartości do wstawienia `{{ }}` | `defaults/`, `group_vars/` |
| **Become** | podniesienie uprawnień (sudo) | `become: true` |
| **Idempotencja** | brak zmian, gdy stan OK | `changed: false` |
| **Handlers** | zadania odpalane "na powiadomienie" | *(jeszcze nie masz)* |

---

## 12. Co dalej (nauka na tym projekcie)

Kolejność nauki, którą polecam — każdy krok bazuje na Twoich plikach:

1. **Uruchom dry-run** `ansible-playbook playbooks/site.yml --check --diff` i
   przeczytaj, co Ansible *chce* zrobić. Nic nie zepsujesz.
2. **Zobacz fakty:** `ansible rpi-01 -m setup | less`. To kopalnia zmiennych,
   których możesz użyć w `when:` i szablonach.
3. **Dodaj własne zadanie** do roli `baseline` (np. zainstaluj `tree`) i
   obserwuj `changed: true` → uruchom ponownie → `changed: false`. Poczujesz
   idempotencję na własnej skórze.
4. **Wdróż klucze SSH** (README, "Next steps") — pozbędziesz się pytań o hasło
   i zrozumiesz, jak zmienne połączenia działają w `group_vars`.
5. **Poznaj handlery** — wzorzec "zmień plik konfiguracyjny → zrestartuj
   usługę, ale tylko jeśli coś się zmieniło". Naturalny następny temat.

Praktyczne rozszerzenie tego wprowadzenia: [demo-obciazenie.md](demo-obciazenie.md)
— pokazuje `host_vars` (zmienne per host), składanie komendy przez `{% if %}` i
tryb `async`/`poll: 0`, na przykładzie różnicowania obciążenia malin.

Oficjalna dokumentacja (bardzo dobra): <https://docs.ansible.com/ansible/latest/>

---

*Ten dokument opisuje stan repo z dnia jego utworzenia. Jak dodasz nowe role
lub playbooki, dopisz je tutaj — dokument najlepiej uczy, gdy odzwierciedla
realny projekt.*
