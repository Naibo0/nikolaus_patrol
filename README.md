# Dokumentation: Haus vom Nikolaus mit TurtleBot3 (ROS 2 Humble)

<img width="5376" height="3200" alt="Haus_vom_Nikolaus2-2" src="https://github.com/user-attachments/assets/cc8a96a7-101a-4972-8b8b-22795bd7d3af" />

Ziel: Wir bauen zwei Programme:
- **Server**: Der Roboter führt die Fahrbewegung aus
- **Client**: Wir geben per Tastatur Befehle (square / triangle / nikolaus)

Wir erstellen ein **eigenes kleines Paket**, damit wir nicht in fremdem Code arbeiten und nichts mit Git “verwirrt”.

---

## Teil 1: Projekt einrichten (ohne Git)

### Schritt 1: Terminal öffnen und ROS laden

Öffne ein Terminal (**STRG+ALT+T**) und lade ROS 2 Humble:

```bash
source /opt/ros/humble/setup.bash
```

---

### Schritt 2: Test-Workspace erstellen

```bash
mkdir -p ~/turtlebot3_test/src
cd ~/turtlebot3_test/src
```

---

### Schritt 3: Eigenes Paket erstellen (unser Projekt)

```bash
ros2 pkg create --build-type ament_python nikolaus_patrol --dependencies rclpy geometry_msgs nav_msgs turtlebot3_msgs
```

Jetzt gibt es u.a. diese Ordner:
- `~/turtlebot3_test/src/nikolaus_patrol/`
- `~/turtlebot3_test/src/nikolaus_patrol/nikolaus_patrol/`

---

### Schritt 4: Vorlage-Dateien herunterladen (nur 2 Dateien)

Wir laden die beiden Beispiel-Dateien direkt von GitHub (ohne git clone):

```bash
cd ~/turtlebot3_test/src/nikolaus_patrol/nikolaus_patrol

wget -O patrol_server.py \
https://raw.githubusercontent.com/ROBOTIS-GIT/turtlebot3/humble/turtlebot3_example/turtlebot3_example/turtlebot3_patrol/turtlebot3_patrol_server.py

wget -O patrol_client.py \
https://raw.githubusercontent.com/ROBOTIS-GIT/turtlebot3/humble/turtlebot3_example/turtlebot3_example/turtlebot3_patrol/turtlebot3_patrol_client.py
```

---

### Schritt 5: Start-Befehle eintragen (setup.py)

Damit wir später `ros2 run ...` benutzen können, müssen wir “Startnamen” registrieren.

Öffne die Datei:

`~/turtlebot3_test/src/nikolaus_patrol/setup.py`

Suche den Bereich `entry_points`. Er sieht ungefähr so aus:

```python
entry_points={
    'console_scripts': [
    ],
},
```

Fülle ihn so aus (kurze Namen, leicht zu tippen):

```python
entry_points={
    'console_scripts': [
        'server = nikolaus_patrol.patrol_server:main',
        'client = nikolaus_patrol.patrol_client:main',
    ],
},
```

Speichern (**STRG+S**).

---

### Schritt 6: Bauen und Umgebung laden

```bash
cd ~/turtlebot3_test
colcon build --symlink-install
source ~/turtlebot3_test/install/setup.bash
```

Test: Sind die Programme registriert?

```bash
ros2 pkg executables nikolaus_patrol
```

Es sollte u.a. erscheinen:
- `nikolaus_patrol server`
- `nikolaus_patrol client`

---

## Teil 2: Programmierung „Haus vom Nikolaus“

### 💡 Wissen: Von Quaternion zu Euler (Warum wir rechnen müssen)

Wenn ihr in den Code schaut, seht ihr die Funktion `get_yaw()`. Warum brauchen wir die?

### 1) Das Format (Quaternion)
Der Roboter liefert uns seine Orientierung in einem Format namens Quaternion (s. Topic odom). Das ist eine fortgeschrittene mathematische Methode mit 4 Werten (x, y, z, w), um Orientierungen im Raum darzustellen. Das ist sehr stabil für Computer, aber für uns Menschen schwer lesbar.

### 2) Das Ziel (Euler-Winkel)
**Wir wollen mit Grad-Zahlen arbeiten** (z.B. 90 Grad Drehung). Diese Darstellung nennt man Euler-Winkel (Roll, Pitch, Yaw).

### 3) Die Umrechnung
Um von den 4 Quaternion-Zahlen auf unseren Winkel zu kommen, nutzen wir eine mathematische Formel mit dem Arkustangens. Im Code seht ihr das hier, die Formel zur Umrechnung von Yaw:

```python
math.atan2(2.0 *(q.w * q.z + q.x * q.y), 1.0 - 2.0 * (q.y * q.y + q.z * q.z))
```
### 4) Warum nur Z-Achse (Yaw)?
Euler-Winkel bestehen eigentlich aus drei Werten:
Roll (Rollen/Kippen zur Seite)
Pitch (Nicken nach vorne/hinten)
Yaw (Gieren/Drehen links-rechts)
Da unser Roboter flach auf dem Boden fährt und nicht wie ein Flugzeug kippt, interessiert uns nur die Drehung um die senkrechte Achse (die Z-Achse). Das ist der Yaw-Winkel.

### Schritt 1: Client ändern (Menü + Taste n)

Datei:
`~/turtlebot3_test/src/nikolaus_patrol/nikolaus_patrol/patrol_client.py`

1) Menü erweitern:

```python
print('Input below')
print('mode: s: square, t: triangle, n: nikolaus')
print('travel_distance (unit: m)')
```

2) Tasteneingabe erweitern:

```python
if mode == 's':
    mode = 1
elif mode == 't':
    mode = 2
elif mode == 'n':
    mode = 3
elif mode == 'x':
    rclpy.shutdown()
```

Speichern (**STRG+S**).

---

### Schritt 2: Server anpassen (Logik)

Datei:  
`~/turtlebot3_test/src/nikolaus_patrol/nikolaus_patrol/patrol_server.py`

---

#### 2.1 WICHTIG: Bewegung optimieren (`go_front`)

In der Original-Datei prüft der Roboter nur **einmal pro Sekunde** seine Position. Das ist zu ungenau und kann dazu führen, dass Linien zu lang/zu kurz werden.

1. Sucht die Funktion `go_front`.
2. Geht in die `while`-Schleife ganz nach unten und ändert `time.sleep(...)`.

**Alt:**
```python
time.sleep(1)
```

**Neu:**
```python
time.sleep(0.1)
```

Das sieht dann z.B. so aus:

```python
def go_front(self, position, length):
        while True:
            # ... (anderer Code) ...

            # WICHTIGE ÄNDERUNG:
            time.sleep(0.1)   # Vorher stand hier 1

        self.init_twist()
```

Speichern (**STRG + S**).

---

#### 2.2 Verteiler erweitern (`execute_callback`)

Sucht die Methode `execute_callback`. Dort wird entschieden, ob Square oder Triangle gefahren wird.  
Fügt den Block für **Modus 3** (Nikolaus) hinzu:

```python
elif self.goal_msg.goal.x == 2:
            # ... (hier steht der Code für Triangle) ...
            break

        # --- NEU EINFÜGEN ---
        elif self.goal_msg.goal.x == 3:
            for count in range(iteration):
                self.nikolaus(feedback_msg, goal_handle, length)
            feedback_msg.state = 'nikolaus patrol complete!!'
            break
        # --------------------
```

Speichern (**STRG + S**).

---

#### 2.3 Die Funktion `nikolaus` implementieren

Fügt ganz am Ende der Klasse (aber **vor** `def main`) eure neue Funktion ein.

Kopiert dieses Gerüst in euren Code:

```python
def nikolaus(self, feedback_msg, goal_handle, length):
        # 1. Geschwindigkeit setzen
        self.linear_x = 0.2
        self.angular_z = 13 * (90.0 / 180.0) * math.pi / 100.0

        # --- EURE AUFGABE STARTET HIER ---

        # Tipp zur Mathematik:
        # Die geraden Wände haben die Länge 'length'.
        # Die Diagonalen und das Dach müsst ihr mit dem Satz des Pythagoras berechnen.

        # WICHTIG ZUM DREHEN:
        # Der Roboter dreht in diesem Skript immer nach LINKS (gegen den Uhrzeigersinn).
        # Eine Drehung von 90 Grad ist einfach.
        # Wenn ihr eigentlich "Rechts" drehen wollt (z.B. -90 Grad), wird der Roboter
        # sich fast einmal komplett im Kreis drehen (270 Grad Links), um das Ziel zu erreichen.
        # PROFI-TIPP: Versucht einen Weg zu finden, bei dem der Roboter nur Linkskurven fährt!

        # Beispiel (Ersetzt die Werte durch eure berechneten Zahlen):
        # distanzen = [length, length, ..., ..., ..., ..., ..., ...]
        # winkel    = [90.0,   90.0,    ..., ..., ..., ..., ..., ...]

        # Die Schleife geht nun Schritt für Schritt durch eure Liste:
        for i in range(8):
            self.position.x = 0.0
            self.angle = 0.0

            # Hier müsst ihr den Roboter bewegen.
            # Nutzt distanzen[i] für die Länge des aktuellen Schritts.
            # Nutzt winkel[i] für die Drehung danach.

            # self.go_front(self.position.x, distanzen[i])
            # ... (hier fehlt noch der Dreh-Befehl!)

            # Feedback senden (Nicht löschen!)
            feedback_msg.state = 'Linie ' + str(i + 1)
            goal_handle.publish_feedback(feedback_msg)
            time.sleep(0.1)

        # --- EURE AUFGABE ENDET HIER ---

        self.init_twist() # Stoppt den Roboter am Ende
```

**Wichtig:**
- Achtet auf die **Einrückung** (Python!).
- Speichert am Ende (**STRG + S**).
- Danach könnt ihr in Teil 3 neu bauen und testen.

## Teil 3: Testen

Nach Änderungen immer neu bauen:

```bash
cd ~/turtlebot3_test
colcon build --symlink-install
source ~/turtlebot3_test/install/setup.bash
```

### Terminal 1 (Server starten)

```bash
source /opt/ros/humble/setup.bash
source ~/turtlebot3_test/install/setup.bash
ros2 run nikolaus_patrol server
```

### Terminal 2 (Client starten)

```bash
source /opt/ros/humble/setup.bash
source ~/turtlebot3_test/install/setup.bash
ros2 run nikolaus_patrol client
```

### Roboter-Umgebung starten (WÄHLT EINE OPTION)
Ihr müsst euch entscheiden: Wollt ihr im Simulator testen oder am echten Roboter?

#### Option A: Simulation (Am PC)
Öffnet ein neues Terminal und startet Gazebo:
```bash
ros2 launch turtlebot3_gazebo empty_world.launch.py
```
#### Option B: Echter Roboter (Im Labor)
Dafür müsst ihr erst den Roboter starten und dann euch per SSH auf dem Roboter einloggen.
(Hinweis: Fragt nach dem Hostnamen(bsp. tb3-b02)!)

Öffnet ein Terminal und verbindet euch:
```bash
# 1. Verbindung herstellen (Passwort meistens: ubuntu oder turtlebot)
ssh ubuntu@tb3-b02.etas.fh-muenster.de

# 2. Roboter-Hardware starten
ros2 launch turtlebot3_bringup robot.launch.py
```

Eingabe:

* Mode: n
* Distance: 0.3 (Am echten Roboter Platz lassen!)
* Count: 1

---
