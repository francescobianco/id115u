# id115u

Controllo di un braccialetto **ID115U** (protocollo IDO / VeryFit) da Linux via Bluetooth LE,
senza l'app ufficiale. Tutto quello che c'è qui è stato verificato su un dispositivo reale.

## Dispositivo

| Campo | Valore |
|---|---|
| Nome | `ID115U` |
| MAC | `E7:84:6B:69:A2:8A` (random) |
| Servizio | `0af0` |
| Device ID | `0x02be` (702) |
| Firmware | 23 (`0x17`) |

## Uso rapido

```shell
./id115u vibrate            # vibra (chiamata in arrivo), annullata dopo 4 s
./id115u vibrate 10         # vibra per 10 s
./id115u notify Mario ciao  # notifica WhatsApp
./id115u time               # legge l'orologio del braccialetto
./id115u settime            # imposta l'orologio all'ora locale
./id115u find               # vibrazione "trova dispositivo" per 5 s
./id115u alarms             # programma le sveglie da alarms.txt
./id115u battery            # batteria: % , mV, stato
./id115u live               # totali di oggi: passi, calorie, distanza, minuti attivi
./id115u activity           # attività di oggi a intervalli di 15 minuti
./id115u info               # info dispositivo (risposta grezza)
./id115u mac                # legge il MAC
./id115u raw 0201           # comando grezzo (padding a 20 byte automatico)
./id115u raw h:0803010000   # scrittura grezza sul canale salute 0af1
./id115u raw read:0x0016    # lettura di una caratteristica
```

Variabili d'ambiente: `MAC`, `RETRIES` (default 5), `CONNECT_TIMEOUT` in secondi (default 20),
`KEEP_LOG=<file>` per salvare il log grezzo di gatttool.

Dipendenze: `gatttool` (bluez-deprecated), `xxd`, GNU `awk`.

## Abbinamento

Se il braccialetto si connette e si disconnette subito, l'abbinamento salvato non è più valido
(es. dopo un reset). Va rimosso e rifatto:

```shell
bluetoothctl remove E7:84:6B:69:A2:8A
bluetoothctl            # poi, nella shell interattiva:
  agent NoInputNoOutput
  default-agent
  scan on               # attendere che compaia ID115U (anche 30-40 s)
  scan off
  pair E7:84:6B:69:A2:8A
  trust E7:84:6B:69:A2:8A
```

`bluetoothctl connect` da solo non basta: la connessione cade prima che i servizi GATT siano
risolti. `gatttool` invece funziona. La connessione è fragile (segnale debole, movimento,
interferenze sulla scrivania) e può richiedere più di 10 s: per questo lo script ritenta.

## Protocollo

Caratteristiche GATT (servizio `0af0`):

| UUID | Handle valore | Proprietà | Uso |
|---|---|---|---|
| `0af6` | `0x000e` | read, write | comandi |
| `0af7` | `0x0010` | read, notify | risposte (CCCD `0x0011`, scrivere `0100`) |
| `0af1` | `0x0016` | read, write | richieste dati salute (scrittura senza padding) |
| `0af2` | `0x0013` | read, notify | dati salute (CCCD `0x0014`, scrivere `0100`) |

Ogni comando su `0af6` è lungo **20 byte** (padding con `00`). Il primo byte è la classe, il
secondo il comando; la risposta arriva come notifica su `0af7` e ripete i primi due byte.

Classi: `01` OTA, `02` GET, `03` SET, `04` bind/unbind, `05` notifiche, `06` controllo app,
`07` eventi dal braccialetto, `08` dati salute, `20` dump stack, `21` log, `aa` factory,
`f0` riavvio/spegnimento.

### Comandi verificati

| Comando | Byte | Risposta / effetto |
|---|---|---|
| Info dispositivo | `02 01` | `02 01 <id LE 2B> <fw> <mode> <stato batt.> <batt. %> …` → es. `be 02 17 01 01 3a` = id `0x02be`, fw 23, in carica, 58% |
| Funzioni supportate | `02 02` | `02 02 5b 0a 8f 01 07 6d 6b 05 0f 06 …` (bitmap, non decodificata) |
| Lettura ora | `02 03` | `02 03 <anno LE 2B> <mese> <giorno> <ora> <min> <sec> <giorno sett.>` |
| Lettura MAC | `02 04` | `02 04 e7 84 6b 69 a2 8a` |
| Batteria | `02 05` | `02 05 <tipo> <mV LE 2B> <stato> <%> …` → es. `00 22 0f 00 3a` = 3874 mV, 58% |
| ? | `02 06` | `02 06 ff ff …` |
| Notifiche (readback) | `02 10` | `02 10 aa 00 00 aa 03 00` |
| Dati live | `02 a0` | `02 a0 <passi u32> <calorie u32> <distanza m u32> <min attivi u32> <battito>` |
| Imposta ora | `03 01 <anno LE 2B> <mese> <giorno> <ora> <min> <sec> <giorno sett.>` | orologio aggiornato |
| Chiamata in arrivo | `05 01 01 01` | il braccialetto vibra |
| Annulla chiamata | `05 02` | vibrazione interrotta |
| Notifica messaggio | `05 03 <tot> <seq> <tipo> <len mittente> <len numero> <len testo> <mittente> <testo>` | notifica mostrata |
| Trova dispositivo | `06 04 00` / `06 04 01` | avvia / ferma la vibrazione |
| Sveglia | `03 02 <id> <stato> <tipo> <ora> <min> <ripetizione> <snooze>` | risponde `03 02` |

### Sveglie

Il braccialetto supporta **10 sveglie** (byte 1 della tabella funzioni `02 02` = `0x0a`).
`./id115u alarms` usa gli slot 1–10 e disattiva anche lo slot 0.

- **ripetizione**: bit 0 = attiva, bit 1…7 = lunedì…domenica (verificato: `0x11` = solo
  giovedì suona di giovedì, `0x88` no). `0x00` = disattivata.
- **stato**: `0x55` e `0xAA` suonano entrambi; lo script usa `0x55`.
- **tipo** (icona): `00` sveglia; la tabella funzioni dichiara anche sonno, sport, medicina,
  personalizzata (codici non verificati). Non è previsto un testo.

La programmazione sta in `alarms.txt`, una riga per giorno:

```
MON 07:00 13:30
FRI 07:00
```

Lo stesso orario in più giorni diventa una sola sveglia con più bit di ripetizione.

Stato batteria: `0` normale, `1` in carica, `2` carica, `3` batteria scarica.
Il giorno della settimana parte da **lunedì = 0**. Il tipo `0x08` è WhatsApp. In un singolo
pacchetto mittente + numero + testo devono stare in 12 byte; testi più lunghi richiedono più
pacchetti (`<tot>` / `<seq>`), non ancora provato.

Tipi notifica (da Gadgetbridge): `01` generico/SMS, `03` WeChat, `06` Facebook, `07` Twitter,
`08` WhatsApp, `09` Messenger, `0a` Instagram, `0b` LinkedIn.

Tutti gli altri `02 xx` (da `00` a `ff`) non rispondono.

### Dati salute (classe `08`, richiesta su `0af1`, risposte su `0af2`)

Richiesta: `08 <key> 01 00 00` (5 byte, senza padding). La risposta è una serie di pacchetti
`08 <key> <seq> <len> <payload…>` chiusa da `08 ee <tipo> 00 00 00`.

| Key | Contenuto | Note |
|---|---|---|
| `01` | inizio sincronizzazione | risponde `08 01 00 00 00 00 00 00` |
| `02` | fine sincronizzazione | risponde `08 02` |
| `03` | attività di oggi | 34 pacchetti, vedi sotto |
| `04` | sonno di oggi | vuoto (2 pacchetti) |
| `05` | storico attività | vuoto |
| `06` | storico sonno | vuoto |
| `07`–`0a` | – | nessuna risposta |

Attività di oggi (`08 03`):
- pacchetto 1: `<anno LE 2B> <mese> <giorno> 00 00 <minuti per campione = 0f> <n. campioni = 60> <n. pacchetti = 22>`
- pacchetto 2: totali `<passi u32> <calorie u32> <distanza u32> <min attivi u32>`
- pacchetti successivi: campioni da 5 byte, 3 per pacchetto, uno ogni 15 minuti da mezzanotte.
  Con `d01 = b0 | b1<<8` ecc.: passi `(d01 >> 2) & 0xfff`, minuti attivi `(d12 >> 6) & 0xf`,
  calorie `(d23 >> 2) & 0x3ff`, distanza `d34 >> 4`.

Il campo battito in `02 a0` vale 0: probabilmente questo modello non ha sensore di battito
(non verificato). Non è stato trovato un comando per leggere l'accelerometro grezzo.

## Fonti

- [Gadgetbridge, supporto ID115](https://codeberg.org/Freeyourgadget/Gadgetbridge) (`devices/id115`, `service/devices/id115`):
  costanti, fetch attività `08 03`, tipi di notifica.
- [toobur-veryfit-research](https://github.com/d3nd3/toobur-veryfit-research) (`A200-PROTOCOL.md`):
  protocollo IDO per un modello più recente; layout di `02 01`, `02 05`, `02 a0`, `06 04`.

Esempio WhatsApp da "Claude" con testo "Ciao":

```
05 03 01 01 08 06 00 04 43 6c 61 75 64 65 43 69 61 6f 00 00
```

### Con gatttool a mano

```shell
gatttool -t random -b E7:84:6B:69:A2:8A -I
  connect
  char-write-req 0x0011 0100
  char-write-req 0x000e 0203000000000000000000000000000000000000
```

## Firmware / OTA

Non provato. I braccialetti IDO di solito usano un SoC Nordic nRF51/nRF52 con DFU OTA: in
teoria il firmware si può riscrivere senza aprirlo, ma un firmware errato può renderlo
irrecuperabile senza accesso SWD.

## Vedi anche

`../id115`: documentazione e script per un braccialetto diverso (protocollo Yoho Sports,
caratteristica `cc06`), non compatibile con questo.
