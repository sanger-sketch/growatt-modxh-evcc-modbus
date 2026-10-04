# growatt-modxh-evcc-modbus

# Growatt MOD-XH Batteriesteuerung per Modbus TCP mit evcc

**TL;DR (EN):** Battery control for a Growatt MOD 10KTL3-XH + APX 5.0P-B1 via a
USR-TCP232-410S RS485-to-TCP gateway and evcc's `growatt-hybrid-tlxh` template.
The commonly documented registers 1100/1101/1102 (SPH/MIX family) do **not**
work on MOD-XH/TL-XH hardware — use registers 3038/3039/3049 instead
(see evcc PR #30403).

## Ausgangslage

- Wechselrichter: Growatt MOD 10KTL3-XH (dreiphasige Hybrid-Baureihe)
- Batterie: Growatt APX 5.0P-B1
- Bisherige Auslesung: Shine-WiFi-X (USB-A-Port)
- Ziel: Aktive Batteriesteuerung (Laden/Entladen erzwingen) über [evcc](https://evcc.io)
- Gateway: USR-TCP232-410S (RS485-zu-Modbus-TCP, schreibfähig)

## Pinbelegung (Kommunikationsstecker des WR)

Der 30-polige Kommunikationsstecker führt mehrere RS485-Paare – nur eins ist
für die externe Steuerung relevant:

| Pin | Bezeichnung | Zweck |
|---|---|---|
| 3/4 | RS485A1/B1 | **Externe Kommunikation/Steuerung – hier anschließen** |
| 5/6 | RS485A3/B3 | Zähler-Kommunikation (Anti-Rückfluss) |
| 7/8 | RS485A2/B2 | Batterie-BMS (intern, nicht anfassen) |
| 17/18 | RS485A4/B4 | Backup-Box |

Der Shine-WiFi-Adapter hängt separat am USB-A-Port, nicht an RS485 – daher
kein Konflikt mit dem externen Gateway.

## Gateway-Konfiguration (USR-TCP232-410S)

- Baudrate 9600, 8N1, kein Flow Control
- Socket A → Work Mode: **TCP Server**, zweites Dropdown: **Modbus TCP**
  (nicht "None" – sonst reiner Transparent-Modus, WR antwortet nicht)
- Lokaler Port: 502

## Der Stolperstein: falsche Registerfamilie

Die weit verbreitete evcc-Doku für Growatt-Hybridwechselrichter nennt für die
einmalige Freischaltung der aktiven Batteriesteuerung die Register
`1100, 1101, 1102` (Werte `0, 5947, 0`, via FC16). Das gilt für die
SPH/MIX-Baureihe – **nicht** für MOD-XH/TL-XH:

mbpoll -m tcp -a 1 -r 1100 -t 4 -0 192.168.x.x 0 5947 0
→ Write output (holding) register failed: Unknown error ...
(Modbus Exception Code 0 auf FC16)


## Die Lösung: Register 3038/3039/3049

Für MOD-XH/TL-XH gilt ein anderes Template
([`growatt-hybrid-tlxh`](https://github.com/evcc-io/evcc/blob/master/templates/definition/meter/growatt-hybrid-tlxh.yaml))
mit anderen Registern. Wichtig: Start- und Endzeit des "Battery first"-Slots
(Register 3038 + 3039) müssen als **ein** 32-Bit-Wert in einer einzigen
FC16-Transaktion geschrieben werden – ein Einzelschreibvorgang auf 3039 wird
mit Exception 0 abgelehnt (siehe
[evcc PR #30403](https://github.com/evcc-io/evcc/pull/30403)).

| Modus | Register 3038/3039 (uint32) | Register 3049 |
|---|---|---|
| normal | `8192` / `5947` (0x2000173B) | `0` |
| hold (Battery-first, kein AC-Laden) | `40960` / `5947` (0xA000173B) | `0` |
| charge (Battery-first + AC-Laden) | `40960` / `5947` (0xA000173B) | `1` |

Test/Freischaltung mit mbpoll:

mbpoll -m tcp -a 1 -r 3038 -t 4 -0 192.168.x.x 40960 5947
mbpoll -m tcp -a 1 -r 3049 -t 4 -0 192.168.x.x 0

Verifizieren:

mbpoll -m tcp -a 1 -r 3038 -c 2 -t 4 -0 -1 192.168.x.x
mbpoll -m tcp -a 1 -r 3049 -c 1 -t 4 -0 -1 192.168.x.x


SoC lesen (Input-Register, FC03/`-t 3`, nicht Holding!):

mbpoll -m tcp -a 1 -r 3171 -c 1 -t 3 -0 -1 192.168.x.x


## evcc-Konfiguration

```yaml
meters:
  - name: my_grid
    type: template
    template: growatt-hybrid-tlxh
    usage: grid
    modbus: tcpip
    id: 1
    host: 192.168.x.x
    port: 502

  - name: my_pv
    type: template
    template: growatt-hybrid-tlxh
    usage: pv
    modbus: tcpip
    id: 1
    host: 192.168.x.x
    port: 502
    maxacpower: 10000

  - name: my_battery
    type: template
    template: growatt-hybrid-tlxh
    usage: battery
    modbus: tcpip
    id: 1
    host: 192.168.x.x
    port: 502
    capacity: 10
```

## Fazit

Wer einen Growatt MOD-XH/TL-XH mit evcc über Modbus TCP steuern will: Finger
weg vom `growatt-hybrid`-Template (Register 1100–1102), stattdessen
`growatt-hybrid-tlxh` (Register 3038/3039/3049) verwenden. Hardware-seitig
mit einem USR-TCP232-410S (Modbus-TCP-Modus) über RS485A1/B1 getestet und
bestätigt.

## Quellen
- https://github.com/evcc-io/evcc/pull/30403
- https://github.com/evcc-io/evcc/issues/23300
- https://docs.evcc.io/de/meters/growatt-hybrid-inverter/
- https://github.com/evcc-io/evcc/blob/master/templates/definition/meter/growatt-hybrid-tlxh.yaml
