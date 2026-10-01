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
./id115u info               # info dispositivo (risposta grezza)
./id115u mac                # legge il MAC
./id115u raw 0201           # comando grezzo (padding a 20 byte automatico)
```

Variabili d'ambiente: `MAC`, `RETRIES` (default 5), `CONNECT_TIMEOUT` in secondi (default 20).

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
| `0af2` | `0x0013` | read, notify | – |
| `0af1` | `0x0016` | read, write | – |

Ogni comando è lungo **20 byte** (padding con `00`). Il primo byte è la classe, il secondo
il comando; la risposta arriva come notifica su `0af7` e ripete i primi due byte.

### Comandi verificati

| Comando | Byte | Risposta / effetto |
|---|---|---|
| Info dispositivo | `02 01` | `02 01 be 02 17 01 00 14 01 01` → id `0x02be`, fw 23 |
| Funzioni supportate | `02 02` | `02 02 5b 0a 8f 01 07 6d 6b 05 0f 06 …` (bitmap, non decodificata) |
| Lettura ora | `02 03` | `02 03 <anno LE 2B> <mese> <giorno> <ora> <min> <sec> <giorno sett.>` |
| Lettura MAC | `02 04` | `02 04 e7 84 6b 69 a2 8a` |
| ? | `02 05` | `02 05 00 77 0e 00 14 00 00 00 00 0d 06 00 00` |
| ? | `02 06` | `02 06 ff ff …` |
| Imposta ora | `03 01 <anno LE 2B> <mese> <giorno> <ora> <min> <sec> <giorno sett.>` | orologio aggiornato |
| Chiamata in arrivo | `05 01 01 01` | il braccialetto vibra |
| Annulla chiamata | `05 02` | vibrazione interrotta |
| Notifica messaggio | `05 03 <tot> <seq> <tipo> <len mittente> <len numero> <len testo> <mittente> <testo>` | notifica mostrata |

Il giorno della settimana parte da **lunedì = 0**. Il tipo `0x08` è WhatsApp. In un singolo
pacchetto mittente + numero + testo devono stare in 12 byte; testi più lunghi richiedono più
pacchetti (`<tot>` / `<seq>`), non ancora provato.

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
