# Demo: zróżnicowane obciążenie floty (dla monitoringu)

Ten dokument opisuje playbook **`playbooks/stress.yml`** oraz profile w
**`inventory/host_vars/`**. Cel demo: wygenerować **inne** obciążenie na każdej
z trzech malin, tak aby na wykresach z `node_exporter` (Prometheus/Grafana) było
widać trzy różne przebiegi — zamiast trzech identycznych linii.

Przy okazji poznajesz najważniejszy mechanizm Ansible do różnicowania hostów:
**`host_vars`** (zmienne *per host*).

---

## 1. Nowe pojęcie: `host_vars` — zmienne dla jednego hosta

Znasz już `group_vars/rpi.yml` — zmienne wspólne dla **całej grupy** (patrz
[wprowadzenie-ansible.md](wprowadzenie-ansible.md), sekcja 5). `host_vars` to ta
sama idea, ale dla **pojedynczego hosta**:

```
inventory/
  hosts.ini                ← lista hostów
  group_vars/rpi.yml       ← zmienne dla CAŁEJ grupy rpi  (to samo dla wszystkich)
  host_vars/
    rpi-01.yml             ← zmienne TYLKO dla rpi-01
    rpi-02.yml             ← zmienne TYLKO dla rpi-02
    rpi-03.yml             ← zmienne TYLKO dla rpi-03
```

Ansible dopasowuje pliki **po nazwie**: `host_vars/rpi-01.yml` automatycznie
przypina się do hosta `rpi-01` z inventory. Nie trzeba niczego podpinać ręcznie
— wystarczy, że katalog `host_vars/` leży obok pliku inventory.

> **Zasada pierwszeństwa:** gdy ta sama zmienna jest i w `group_vars`, i w
> `host_vars`, wygrywa `host_vars` (bardziej szczegółowy = ważniejszy). Dzięki
> temu ustawiasz wspólne rzeczy w grupie, a wyjątki nadpisujesz per host.

Sprawdzić, jakie zmienne widzi Ansible dla danego hosta, możesz bez łączenia się
z malinami:

```bash
ansible-inventory --host rpi-01
```

---

## 2. Profile obciążenia (`inventory/host_vars/*.yml`)

Każdy plik definiuje jedną zmienną — słownik `load_profile` — opisujący, jak
mocno obciążyć każdy z czterech zasobów:

```yaml
# inventory/host_vars/rpi-03.yml  — malina "dyskowo-sieciowa"
load_profile:
  cpu: 1            # workery CPU
  vm: 1             # workery pamięci...
  vm_bytes: "15%"   # ...ile RAM alokuje każdy (procent całości)
  hdd: 2            # workery I/O na dysku
  hdd_bytes: "256M" # ile zapisuje każdy worker dyskowy
  net: 4            # workery ruchu sieciowego (gniazda)
```

Rozkład ról w demo — celowo **każda malina ma inny dominujący zasób**, żeby
przebiegi łatwo było od siebie odróżnić:

| host   | rola w demo        | CPU | RAM       | dysk I/O    | sieć  |
|--------|--------------------|-----|-----------|-------------|-------|
| rpi-01 | „procesorowa"      | 4 (pełne) | lekko (15%) | — (wył.) | lekko |
| rpi-02 | „pamięciowa"       | tło | ~50% RAM  | — (wył.)    | lekko |
| rpi-03 | „dyskowo-sieciowa" | tło | lekko     | 2× (256M)   | mocno |

Żeby zmienić demo, edytujesz **dane** (te pliki), a nie **logikę** (playbook) —
dokładnie ta sama filozofia co przy `defaults/` w rolach.

---

## 3. Playbook (`playbooks/stress.yml`)

Playbook robi trzy rzeczy: instaluje `stress-ng`, uruchamia je z profilem hosta,
i wypisuje, co na którym hoście poszło.

### 3a. Instalacja narzędzia

```yaml
- name: Zainstaluj stress-ng
  ansible.builtin.apt:
    name: stress-ng
    state: present
    update_cache: true
```

`stress-ng` to standardowe narzędzie do generowania syntetycznego obciążenia —
jednym poleceniem potrafi obciążyć CPU, pamięć, dysk i sieć.

### 3b. Uruchomienie wg profilu — komenda składana warunkowo

```yaml
- name: Uruchom stress-ng wg profilu hosta (async, fire-and-forget)
  ansible.builtin.command:
    cmd: >-
      stress-ng
      {% if load_profile.cpu | int > 0 %} --cpu {{ load_profile.cpu }}{% endif %}
      {% if load_profile.vm  | int > 0 %} --vm {{ load_profile.vm }} --vm-bytes {{ load_profile.vm_bytes }}{% endif %}
      {% if load_profile.hdd | int > 0 %} --hdd {{ load_profile.hdd }} --hdd-bytes {{ load_profile.hdd_bytes }} --temp-path /tmp{% endif %}
      {% if load_profile.net | int > 0 %} --sock {{ load_profile.net }}{% endif %}
      --timeout {{ duration }}s
  async: "{{ duration | int + 30 }}"
  poll: 0
```

Trzy rzeczy warte uwagi:

1. **Komenda składana z `{% if %}`.** To Jinja2 (ten sam silnik co `{{ }}`).
   Flaga danego zasobu trafia do komendy **tylko** gdy w profilu wartość > 0.
   Powód → punkt niżej.

2. **Pułapka stress-ng: `0` znaczy „tyle, ile rdzeni".** W `stress-ng` liczba
   `0` (np. `--hdd 0`) **nie** oznacza „wyłącz", tylko „uruchom tyle workerów,
   ile jest rdzeni CPU". Gdybyśmy ślepo wstawili `--hdd 0`, obciążylibyśmy dysk
   zamiast go pominąć. Dlatego przy wartości 0 **pomijamy flagę** przez `{% if %}`.

3. **`async` + `poll: 0` = „odpal i zapomnij".** Normalnie Ansible czeka, aż
   zadanie się skończy. Tu obciążenie ma trwać 5 minut — nie chcemy blokować na
   ten czas. `poll: 0` mówi: *wystartuj zadanie na malinie w tle i natychmiast
   wróć*. `async` to górny limit czasu życia zadania (dajemy `duration + 30 s`
   zapasu). `stress-ng` sam się kończy po `--timeout`.

### 3c. Zmienna `duration`

```yaml
vars:
  duration: 300   # sekundy
```

Domyślnie 5 minut. Nadpiszesz z linii komend flagą `-e` (extra vars — patrz
wprowadzenie, sekcja 9):

```bash
ansible-playbook playbooks/stress.yml -e duration=120   # 2 minuty
```

---

## 4. Jak uruchomić i obserwować

```bash
cd ~/3xRPi-ops

# start obciążenia (poda się hasło SSH i BECOME)
ansible-playbook playbooks/stress.yml

# krócej
ansible-playbook playbooks/stress.yml -e duration=120
```

Playbook odda znak zachęty **od razu** (dzięki `poll: 0`), a obciążenie kręci się
w tle na malinach. W tym czasie patrz w Grafanę / metryki `node_exporter`:

- **CPU** — `rpi-01` powinno wyskoczyć do ~100%.
- **RAM** — `rpi-02` powinno pokazać ~50% zajętej pamięci.
- **dysk I/O** — `rpi-03` powinno mieć aktywność zapisu.

### Zatrzymanie przed czasem

`stress-ng` samo kończy po `--timeout`, ale możesz przerwać wcześniej komendą
ad-hoc:

```bash
ansible rpi -b -a "pkill -f stress-ng"
```

(`-b` = become/sudo, bo proces może działać jako root po instalacji przez playbook.)

---

## 5. Znane ograniczenia

- **Sieć = na razie loopback.** `stress-ng --sock` generuje ruch po interfejsie
  `lo`, a **nie** po fizycznej karcie (`eth0`/`wlan0`). W `node_exporter`
  zobaczysz go na urządzeniu `lo`. Jeśli chcesz **realny ruch między malinami**
  (widoczny na fizycznym interfejsie), następnym krokiem jest `iperf3`: serwer na
  jednej malinie, klienci na pozostałych. To osobne, nieco bardziej złożone demo.

- **Karta SD.** Obciążenie dysku jest włączone tylko na `rpi-03`, celowo małe
  (`256M` na worker) i krótkie. Karty SD zużywają się od zapisów — **nie
  zostawiaj tego demo w pętli na godziny.**

- **RAM a OOM.** Profile pamięci trzymają się ≤ ~50% RAM, żeby nie wywołać
  zabójcy OOM (out-of-memory). Jeśli podniesiesz `vm_bytes`, rób to ostrożnie i
  pamiętaj, że przy `vm: 2` całość to `2 × vm_bytes`.

---

## 6. Czego się tu nauczyłeś

| Pojęcie | Co to | Gdzie |
|---|---|---|
| **`host_vars`** | zmienne dla pojedynczego hosta | `inventory/host_vars/*.yml` |
| **Pierwszeństwo zmiennych** | `host_vars` bije `group_vars` | — |
| **Komenda składana Jinja** | budowanie argumentów przez `{% if %}` | `stress.yml` |
| **`async` + `poll: 0`** | „odpal i zapomnij", zadanie w tle | `stress.yml` |
| **`ansible-inventory --host`** | podgląd zmiennych hosta bez łączenia | debugowanie |

Naturalny następny krok: przerobić „sieć" na realny ruch między malinami przez
`iperf3`.
