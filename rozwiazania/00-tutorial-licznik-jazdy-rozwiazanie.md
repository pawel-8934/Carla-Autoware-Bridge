# Tutorial: Licznik jazdy — rozwiązanie

**Czas:** 1 zajęcia · **Praca:** samodzielnie, krok po kroku · **Wymaga:** Python, przejście
[README repozytorium](../README.md) do przejazdu do celu włącznie

> **Jak wypełniać ten plik**
>
> Przy każdym punkcie **Sprawdź** są oznaczone pola do uzupełnienia:
>
> - `✏️ ODPOWIEDŹ` — wpisz tekst odpowiedzi w miejscu zaznaczonym `_…_`
> - `📷 SCREEN` — wklej zrzut ekranu (lub ścieżkę do pliku) w oznaczonym bloku
>
> Nie usuwaj oznaczeń — wypełniaj tylko treść pod nimi.

Zbudujesz licznik jazdy jak w samochodzie: aktualna prędkość, przejechany dystans,
prędkość maksymalna, tryb pracy Autoware i czas jazdy autonomicznej. Najpierw jako log
w terminalu, potem w oknie.

Po drodze poznasz wszystko, czego potrzeba w [projekcie 1](../projekty/01-panel-kierowcy.md):
narzędzia `ros2` w terminalu, własny pakiet, subskrypcje, timer, czas symulacji, QoS tematów
Autoware i okno PyQt5 połączone z ROS. Każdy krok kończy się punktem **Sprawdź**: nie idź dalej,
dopóki nie działa.

## Pięć pojęć na start

| Pojęcie | Co to | Przykład |
|---|---|---|
| **węzeł** (node) | program podłączony do ROS. Autoware to ok. 170 węzłów | `velocity_smoother` |
| **temat** (topic) | nazwany kanał, którym węzły wysyłają sobie dane | `/vehicle/status/velocity_status` |
| **wiadomość** (message) | jedna porcja danych o ustalonym typie (polach) | `VelocityReport` z polem `longitudinal_velocity` |
| **subskrypcja** | „chcę dostawać wiadomości z tematu X”: ROS woła Twoją funkcję po każdej z nich | `create_subscription(...)` |
| **pakiet** | katalog z węzłami, który `colcon` buduje i instaluje | `carla_autoware_examples` |

## Krok 1. Rozejrzyj się w działającym Autoware

Uruchom Autoware (`carla-aw`) jak w README i zostaw je włączone. Wszystko poniżej wpisujesz
w **drugim** terminalu: `./scripts/aw_shell.sh` na hoście.

```bash
ros2 node list | wc -l                          # ile węzłów działa
ros2 topic list | grep vehicle                  # tematy ze stanem pojazdu
ros2 topic info /vehicle/status/velocity_status # typ wiadomości i liczba nadawców/odbiorców
ros2 topic hz /vehicle/status/velocity_status   # jak często przychodzi (Ctrl+C kończy)
ros2 topic echo /vehicle/status/velocity_status # podgląd na żywo (Ctrl+C kończy)
ros2 interface show autoware_vehicle_msgs/msg/VelocityReport   # jakie pola ma wiadomość
```

Teraz temat z trybem pracy Autoware:

```bash
ros2 topic echo /api/operation_mode/state
ros2 interface show autoware_adapi_v1_msgs/msg/OperationModeState
```

**Sprawdź** (zapisz odpowiedzi, przydadzą się):

1. W jakich jednostkach jest `longitudinal_velocity`? (podpowiedź: komentarz w `interface show`)

   > ✏️ **ODPOWIEDŹ 1.1**
   >
   > _Tu wpisz jednostki (np. m/s):_

2. Ile razy na sekundę przychodzi prędkość?

   > ✏️ **ODPOWIEDŹ 1.2**
   >
   > _Tu wpisz przybliżoną częstotliwość (Hz):_

3. Jaką liczbą jest zapisany tryb AUTONOMOUS?

   > ✏️ **ODPOWIEDŹ 1.3**
   >
   > _Tu wpisz liczbę:_

4. Przejedź do celu (RViz albo `drive_to_goal`) z otwartym `ros2 topic echo /api/operation_mode/state`.
   Kiedy przychodzą nowe wiadomości: regularnie czy tylko przy zmianie?

   > ✏️ **ODPOWIEDŹ 1.4**
   >
   > _Tu wpisz obserwację (regularnie / tylko przy zmianie):_

   > 📷 **SCREEN 1.4** — terminal z `ros2 topic echo /api/operation_mode/state` podczas zmiany trybu
   >
   > _Wklej zrzut ekranu poniżej:_
   >
   > <!-- wklej obrazek: ![opis](sciezka/do/screenu.png) -->

## Krok 2. Własny pakiet

W terminalu kontenera:

```bash
cd ~/ws/src
ros2 pkg create licznik_jazdy --build-type ament_python --license Apache-2.0 \
    --dependencies rclpy autoware_vehicle_msgs autoware_adapi_v1_msgs
```

Nazwa pakietu musi stać **przed** `--dependencies`: wszystko po tej opcji jest traktowane jako
zależność.

Powstał katalog `ws/src/licznik_jazdy/` (na hoście też: otwórz go w VS Code):

```
licznik_jazdy/
├── package.xml          # nazwa, opis, zależności
├── setup.py             # jak zainstalować pakiet i jakie komendy udostępnia
├── licznik_jazdy/       # tu będą Twoje pliki .py
│   └── __init__.py
├── resource/  test/  setup.cfg  LICENSE
```

**Sprawdź:** pakiet się buduje.

```bash
cd ~/ws && colcon build --symlink-install --packages-select licznik_jazdy
```

Na końcu ma być `Summary: 1 package finished`.

> 📷 **SCREEN 2** — udany build (`Summary: 1 package finished`)
>
> _Wklej zrzut ekranu poniżej:_
>
> <!-- wklej obrazek: ![build licznik_jazdy](sciezka/do/screenu.png) -->

## Krok 3. Pierwszy węzeł: wypisz prędkość

Utwórz plik `licznik_jazdy/licznik_jazdy/predkosc.py`:

```python
from autoware_vehicle_msgs.msg import VelocityReport
import rclpy
from rclpy.node import Node


class Predkosc(Node):

    def __init__(self):
        super().__init__('predkosc')
        # (typ wiadomości, nazwa tematu, funkcja wołana po każdej wiadomości, długość kolejki)
        self.create_subscription(
            VelocityReport, '/vehicle/status/velocity_status', self.on_velocity, 10)

    def on_velocity(self, msg):
        kmh = msg.longitudinal_velocity * 3.6
        self.get_logger().info(f'prędkość: {kmh:.1f} km/h')


def main():
    rclpy.init()
    node = Predkosc()
    try:
        rclpy.spin(node)    # czeka na wiadomości i woła on_velocity, aż do Ctrl+C
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

Zwróć uwagę: nigdzie nie wołasz `on_velocity` sam. Robi to `rclpy.spin()` za każdym razem,
gdy przyjdzie wiadomość.

Żeby `ros2 run` znalazł węzeł, dopisz go w `setup.py` w `entry_points`
(format: `nazwa_komendy = pakiet.plik:funkcja`):

```python
    entry_points={
        'console_scripts': [
            'predkosc = licznik_jazdy.predkosc:main',
        ],
    },
```

Zbuduj i uruchom:

```bash
cd ~/ws && colcon build --symlink-install --packages-select licznik_jazdy
source install/setup.bash
ros2 run licznik_jazdy predkosc --ros-args -p use_sim_time:=true
```

`source install/setup.bash` jest potrzebne raz w każdym terminalu po zbudowaniu **nowego**
pakietu. Nowo otwierane terminale robią to same.

**Sprawdź:** ok. 20 linii na sekundę `prędkość: 0.0 km/h`, a w czasie jazdy rosnące wartości.
W drugim terminalu `ros2 node list | grep predkosc` pokazuje Twój węzeł.

> 📷 **SCREEN 3** — log węzła `predkosc` z wartościami prędkości (najlepiej w trakcie jazdy)
>
> _Wklej zrzut ekranu poniżej:_
>
> <!-- wklej obrazek: ![predkosc](sciezka/do/screenu.png) -->

> ✏️ **ODPOWIEDŹ 3** — czy `ros2 node list | grep predkosc` pokazał węzeł?
>
> _Tak / Nie (+ ewentualny komentarz):_

## Krok 4. Licznik: stan, timer i czas symulacji

Wypisywanie 20 razy na sekundę jest nieczytelne. Licznik będzie zbierał dane w jednym obiekcie
stanu, a wypisywał je raz na sekundę. Utwórz `licznik_jazdy/licznik_jazdy/licznik.py`:

```python
from dataclasses import dataclass

from autoware_adapi_v1_msgs.msg import OperationModeState
from autoware_vehicle_msgs.msg import VelocityReport
import rclpy
from rclpy.node import Node
from rclpy.qos import DurabilityPolicy, QoSProfile, ReliabilityPolicy

# Tematy /api/... publikują stan raz, przy zmianie, i zostawiają go dla spóźnionych
# subskrybentów ("transient local"). Żeby go dostać, subskrybent musi o to poprosić.
API_QOS = QoSProfile(
    depth=1,
    reliability=ReliabilityPolicy.RELIABLE,
    durability=DurabilityPolicy.TRANSIENT_LOCAL,
)

# Liczba -> nazwa, np. 2 -> 'AUTONOMOUS'. Stałe są zdefiniowane w samej wiadomości.
NAZWY_TRYBOW = {
    OperationModeState.UNKNOWN: 'UNKNOWN',
    OperationModeState.STOP: 'STOP',
    OperationModeState.AUTONOMOUS: 'AUTONOMOUS',
    OperationModeState.LOCAL: 'LOCAL',
    OperationModeState.REMOTE: 'REMOTE',
}


@dataclass
class StanLicznika:
    predkosc_kmh: float | None = None   # None = jeszcze nic nie przyszło
    tryb: int | None = None
    dystans_m: float = 0.0
    max_predkosc_kmh: float = 0.0
    czas_autonomiczny_s: float = 0.0


def tekst(wartosc, wzor='{:.1f}'):
    """Liczba jako tekst albo '—', gdy jeszcze jej nie ma."""
    return '—' if wartosc is None else wzor.format(wartosc)


class Licznik(Node):

    def __init__(self, nazwa='licznik_jazdy'):
        super().__init__(nazwa)
        self.stan = StanLicznika()
        self.ostatni_czas = None

        self.create_subscription(
            VelocityReport, '/vehicle/status/velocity_status', self.on_velocity, 10)
        self.create_subscription(
            OperationModeState, '/api/operation_mode/state', self.on_mode, API_QOS)

    def on_velocity(self, msg):
        s = self.stan
        # abs(): na postoju Autoware raportuje czasem -0.0, a licznik nie musi znać kierunku.
        s.predkosc_kmh = abs(msg.longitudinal_velocity) * 3.6
        s.max_predkosc_kmh = max(s.max_predkosc_kmh, s.predkosc_kmh)

        # Czas z zegara węzła: przy use_sim_time:=true to czas symulacji.
        teraz = self.get_clock().now()
        if self.ostatni_czas is not None:
            dt = (teraz - self.ostatni_czas).nanoseconds / 1e9
            s.dystans_m += abs(msg.longitudinal_velocity) * dt   # droga = prędkość × czas
            if s.tryb == OperationModeState.AUTONOMOUS:
                s.czas_autonomiczny_s += dt
        self.ostatni_czas = teraz

    def on_mode(self, msg):
        self.stan.tryb = msg.mode

    def podsumowanie(self):
        s = self.stan
        return (f'v={tekst(s.predkosc_kmh)} km/h, max={s.max_predkosc_kmh:.1f} km/h, '
                f'dystans={s.dystans_m:.0f} m, tryb={NAZWY_TRYBOW.get(s.tryb, "—")}, '
                f'autonomicznie={s.czas_autonomiczny_s:.0f} s')


def main():
    rclpy.init()
    node = Licznik()
    node.create_timer(1.0, lambda: node.get_logger().info(node.podsumowanie()))
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.try_shutdown()
```

Co tu nowego:

- **Obiekt stanu** (`StanLicznika`). Funkcje subskrypcji tylko zapisują do niego dane,
  a wypisuje je ktoś inny (timer, a w kroku 6 okno). `None` oznacza „jeszcze nie wiem” i wtedy
  wypisujemy „—” zamiast się wywracać.
- **Timer.** `create_timer(1.0, f)` woła `f` co sekundę. Też w ramach `rclpy.spin()`.
- **Dystans** to suma `prędkość × czas` między kolejnymi wiadomościami. Czas bierzemy z zegara
  węzła. Z `use_sim_time:=true` to **czas symulacji** z tematu `/clock`, który płynie tak szybko,
  jak naprawdę liczy się symulacja.
- **Dwie subskrypcje z różnym QoS.** Wyjaśnienie w kroku 5.

Dopisz komendę w `setup.py` (obok `predkosc`):

```python
            'licznik = licznik_jazdy.licznik:main',
```

i zbuduj ponownie (nowa komenda w `setup.py` zawsze wymaga `colcon build`):

```bash
cd ~/ws && colcon build --symlink-install --packages-select licznik_jazdy
ros2 run licznik_jazdy licznik --ros-args -p use_sim_time:=true
```

**Sprawdź:**

1. Zaraz po starcie: `tryb=STOP` (albo AUTONOMOUS, jeśli auto jedzie), a nie `—`.

   > ✏️ **ODPOWIEDŹ 4.1** — jaki tryb zobaczyłeś zaraz po starcie?
   >
   > _STOP / AUTONOMOUS / inne:_

2. Przejedź do celu na tej samej ulicy, przed najbliższym skrzyżowaniem. Dystans w logu
   odpowiada odległości do celu (przykład: cel 150 m dalej → licznik pokazał 151 m),
   a `autonomicznie` rośnie tylko w trybie AUTONOMOUS.

   > ✏️ **ODPOWIEDŹ 4.2** — dystans w liczniku vs odległość do celu
   >
   > _np. cel ≈ … m, licznik pokazał … m:_

   > 📷 **SCREEN 4.2** — log licznika w trakcie / po przejeździe (widać dystans, tryb, autonomicznie)
   >
   > _Wklej zrzut ekranu poniżej:_
   >
   > <!-- wklej obrazek: ![licznik przejazd](sciezka/do/screenu.png) -->

3. Po dojeździe tryb sam zmienia się na STOP.

   > ✏️ **ODPOWIEDŹ 4.3** — czy po dojeździe tryb zmienił się na STOP?
   >
   > _Tak / Nie:_

## Krok 5. Eksperyment: po co ten QoS?

Tryb pracy Autoware zmienia się rzadko, więc jest publikowany tylko przy zmianie. Żeby węzeł
uruchomiony później też go znał, Autoware publikuje go jako *transient local*: ostatnia wiadomość
jest przechowywana i wysyłana każdemu nowemu subskrybentowi, ale tylko takiemu, który o nią
poprosi tym samym QoS.

1. W `licznik.py` zamień w subskrypcji trybu `API_QOS` na `10` (zwykły QoS z kolejką 10).
2. Uruchom licznik ponownie. Budować nie trzeba: dzięki `--symlink-install` zmiana w istniejącym
   pliku `.py` działa od razu.
3. Obserwuj `tryb=—`. Zmieni się dopiero, gdy tryb się zmieni (np. kliknij Auto w RViz).
4. Przywróć `API_QOS`.

`ros2 topic echo` z kroku 1 pokazywał stan od razu, bo sam dobiera QoS do nadawcy. Twój węzeł
tego nie robi. Zasada: **tematy `/api/...` zawsze subskrybuj z `API_QOS`**. Sprawdzisz to
poleceniem `ros2 topic info -v <temat>` (pole `Durability`).

> ✏️ **ODPOWIEDŹ 5** — co zobaczyłeś po zamianie `API_QOS` na `10`?
>
> _Opisz: tryb zaraz po starcie, kiedy się zaktualizował:_

> 📷 **SCREEN 5** — log z `tryb=—` (bez właściwego QoS) albo wynik `ros2 topic info -v /api/operation_mode/state` (Durability)
>
> _Wklej zrzut ekranu poniżej:_
>
> <!-- wklej obrazek: ![qos](sciezka/do/screenu.png) -->

## Krok 6. Okno

Teraz to samo w oknie. Jest jeden problem: `rclpy.spin()` nigdy nie wraca, a okno też ma
swoją pętlę (`app.exec_()`), która musi działać w głównym wątku. Dlatego ROS działa
w **osobnym wątku**, a okno co 100 ms tylko czyta obiekt stanu.

Utwórz `licznik_jazdy/licznik_jazdy/okno.py`:

```python
import signal
import sys
import threading

from licznik_jazdy.licznik import Licznik, NAZWY_TRYBOW, tekst
from PyQt5.QtCore import QTimer
from PyQt5.QtWidgets import QApplication, QLabel, QVBoxLayout, QWidget
import rclpy
from rclpy.executors import SingleThreadedExecutor


class Okno(QWidget):

    def __init__(self, node):
        super().__init__()
        self.node = node
        self.setWindowTitle('Licznik jazdy')
        self.setStyleSheet('font-size: 20px')

        self.predkosc = QLabel()
        self.predkosc.setStyleSheet('font-size: 64px; font-weight: bold')
        self.tryb = QLabel()
        self.dystans = QLabel()
        self.autonomicznie = QLabel()

        uklad = QVBoxLayout(self)
        for etykieta in (self.predkosc, self.tryb, self.dystans, self.autonomicznie):
            uklad.addWidget(etykieta)

        # Co 100 ms przepisz stan z węzła do etykiet.
        self.timer = QTimer(self)
        self.timer.timeout.connect(self.odswiez)
        self.timer.start(100)

    def odswiez(self):
        s = self.node.stan
        self.predkosc.setText(f'{tekst(s.predkosc_kmh, "{:.0f}")} km/h')
        self.tryb.setText(f'Tryb: {NAZWY_TRYBOW.get(s.tryb, "—")}')
        self.dystans.setText(f'Dystans: {s.dystans_m:.0f} m')
        self.autonomicznie.setText(f'Jazda autonomiczna: {s.czas_autonomiczny_s:.0f} s')


def main():
    rclpy.init()
    node = Licznik('licznik_jazdy_okno')   # inna nazwa niż węzeł z licznik.py!

    # ROS w osobnym wątku: spin() czeka na wiadomości i nie wraca, więc w głównym wątku
    # zablokowałby okno. Własny executor, żeby na końcu móc go zatrzymać.
    executor = SingleThreadedExecutor()
    executor.add_node(node)
    watek = threading.Thread(target=executor.spin)
    watek.start()

    app = QApplication(sys.argv)
    okno = Okno(node)
    okno.show()
    signal.signal(signal.SIGINT, lambda *_: app.quit())   # Ctrl+C zamyka okno
    app.exec_()

    # Po zamknięciu okna: zatrzymaj wątek ROS i dopiero wtedy sprzątaj.
    executor.shutdown()
    watek.join()
    node.destroy_node()
    rclpy.try_shutdown()
```

Okno używa tej samej klasy `Licznik` co krok 4. Nic nie trzeba pisać drugi raz.

Trzy rzeczy, które łatwo zepsuć:

- **Kolejność na końcu.** Najpierw `executor.shutdown()` i `watek.join()`, dopiero potem
  `destroy_node()`. Odwrotnie program kończy się błędem `Segmentation fault`.
- **Nie zastępuj wątku wywołaniem `rclpy.spin_once()` w timerze okna.** Obsługuje ono jedną
  wiadomość na wywołanie, czyli 10 na sekundę przy timerze 100 ms, a sama prędkość przychodzi
  20 razy na sekundę. Licznik by się spóźniał coraz bardziej.
- **Ctrl+C.** Bez linii z `signal.signal(...)` okno Qt nie zamyka się po Ctrl+C w terminalu.

Dopisz `'okno = licznik_jazdy.okno:main',` w `setup.py`, zbuduj i uruchom:

```bash
cd ~/ws && colcon build --symlink-install --packages-select licznik_jazdy
ros2 run licznik_jazdy okno --ros-args -p use_sim_time:=true
```

Okno otwiera się na ekranie hosta tak samo jak RViz. Jeśli zamiast okna pojawia się błąd
`could not connect to display`, uruchom na hoście `xhost +local:docker`.

**Sprawdź:** okno pokazuje te same wartości co log z kroku 4 (możesz uruchomić oba naraz)
i zamyka się czysto, bez błędów, zarówno krzyżykiem, jak i przez Ctrl+C.

> 📷 **SCREEN 6** — działające okno licznika (najlepiej w trakcie jazdy / po przejeździe)
>
> _Wklej zrzut ekranu poniżej:_
>
> <!-- wklej obrazek: ![okno licznika](sciezka/do/screenu.png) -->

> ✏️ **ODPOWIEDŹ 6** — czy okno zamyka się czysto (krzyżyk i Ctrl+C), bez błędów w terminalu?
>
> _Tak / Nie (+ ewentualny komunikat błędu):_

## Krok 7. Dwa błędy, które warto zobaczyć raz

**Ta sama nazwa węzła dwa razy.** Uruchom `ros2 run licznik_jazdy licznik` w dwóch terminalach
naraz. `ros2 node list | sort | uniq -d` pokaże `/licznik_jazdy`. Teraz spróbuj włączyć tryb
autonomiczny w RViz: **nie da się**. Autoware traktuje powtórzoną nazwę jako błąd systemu
(`duplicated_node_checker`). Zamknij jeden z liczników i Auto znów działa. Drugą kopię
można uruchomić pod inną nazwą:

```bash
ros2 run licznik_jazdy licznik --ros-args -p use_sim_time:=true -r __node:=licznik_2
```

> ✏️ **ODPOWIEDŹ 7.1** — co się stało z przyciskiem Auto przy dwóch węzłach o tej samej nazwie?
>
> _Opisz obserwację:_

> 📷 **SCREEN 7.1** — `ros2 node list | sort | uniq -d` albo nieaktywny Auto w RViz
>
> _Wklej zrzut ekranu poniżej:_
>
> <!-- wklej obrazek: ![zduplikowany wezel](sciezka/do/screenu.png) -->

**Brak `use_sim_time`.** Uruchom obok siebie licznik z `use_sim_time:=true` i bez tej opcji
(pod inną nazwą, jak wyżej) i przejedź trasę. Bez czasu symulacji węzeł liczy czas zegarem
komputera. Póki symulacja nadąża za czasem rzeczywistym, wyniki są takie same. Na wspólnym
serwerze, gdy symulacja zwalnia, licznik bez czasu symulacji pokazuje za duży dystans: liczy
prędkość z symulacji razy czas z prawdziwego zegara. Zasada: **zawsze `use_sim_time:=true`**.

> ✏️ **ODPOWIEDŹ 7.2** — porównanie dystansu / czasu: z `use_sim_time:=true` vs bez
>
> _Opisz różnicę (lub brak różnicy, jeśli symulacja nie zwalniała):_

> 📷 **SCREEN 7.2** _(opcjonalnie)_ — logi obu liczników obok siebie
>
> _Wklej zrzut ekranu poniżej:_
>
> <!-- wklej obrazek: ![use_sim_time](sciezka/do/screenu.png) -->

## Gdy coś nie działa

| Objaw | Przyczyna |
|---|---|
| `Package 'licznik_jazdy' not found` | brak `source install/setup.bash` po pierwszym buildzie |
| `No executable found` | komenda nie dopisana w `setup.py` albo brak `colcon build` po dopisaniu |
| `ros2 pkg create: error: ... package_name` | nazwa pakietu po `--dependencies` (krok 2) |
| węzeł działa, ale nic nie wypisuje | literówka w nazwie tematu (porównaj z `ros2 topic list`) albo Autoware nie działa |
| `tryb=—` mimo działającego Autoware | brak `API_QOS` przy temacie `/api/...` (krok 5) |
| przycisk Auto w RViz nieaktywny | dwa węzły o tej samej nazwie (krok 7) |
| dziwne czasy i dystanse | brak `use_sim_time:=true` |
| `could not connect to display` | na hoście: `xhost +local:docker` |
| `Segmentation fault` przy zamykaniu okna | zła kolejność sprzątania (krok 6) |

## Od licznika do panelu kierowcy

W [projekcie 1](../projekty/01-panel-kierowcy.md) robisz to samo na większą skalę:

| W tutorialu | W projekcie 1 |
|---|---|
| 2 tematy | 6 tematów, w tym trzy `/api/...` i limit prędkości (wszystkie z `API_QOS`) |
| `StanLicznika` | obiekt stanu z pozycją auta i celem trasy, odległość do celu |
| log co sekundę | to samo (zadanie 1) |
| okno z etykietami | kolory: czerwony licznik po przekroczeniu limitu, kolor trybu, komunikat „Przejmij sterowanie” |

Typ i pola nowych tematów sprawdzisz tak jak w kroku 1: `ros2 topic info -v`
i `ros2 interface show`.

## Checklist oddania

Upewnij się, że uzupełniłeś wszystkie pola:

| Nr | Typ | Opis | Uzupełnione? |
|---|---|---|---|
| 1.1 | ✏️ | jednostki `longitudinal_velocity` | ☐ |
| 1.2 | ✏️ | częstotliwość prędkości | ☐ |
| 1.3 | ✏️ | liczba trybu AUTONOMOUS | ☐ |
| 1.4 | ✏️ + 📷 | kiedy przychodzą wiadomości trybu | ☐ |
| 2 | 📷 | udany `colcon build` | ☐ |
| 3 | 📷 + ✏️ | działający węzeł `predkosc` | ☐ |
| 4.1 | ✏️ | tryb zaraz po starcie | ☐ |
| 4.2 | ✏️ + 📷 | dystans po przejeździe | ☐ |
| 4.3 | ✏️ | STOP po dojeździe | ☐ |
| 5 | ✏️ + 📷 | eksperyment QoS | ☐ |
| 6 | 📷 + ✏️ | okno + czyste zamknięcie | ☐ |
| 7.1 | ✏️ + 📷 | zduplikowana nazwa węzła | ☐ |
| 7.2 | ✏️ (+ 📷) | brak `use_sim_time` | ☐ |
