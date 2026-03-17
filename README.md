# 📡 Guida Completa: Ricezione SNMP Traps su Zabbix 7.0 (via Perl Script)

Questa guida illustra passo dopo passo come configurare un server Zabbix 7.0 per ricevere, processare e associare le Trap SNMP provenienti dai dispositivi di rete, utilizzando lo script Perl ufficiale di Zabbix.

## 🏗️ Architettura del flusso
Comprendere come viaggia l'informazione è fondamentale per il troubleshooting:
1. **Dispositivo di Rete** genera una Trap SNMP e la invia via rete (UDP/162).
2. Il demone **`snmptrapd`** (in esecuzione sul server Zabbix) riceve la trap.
3. **`snmptrapd`** la passa allo script **`zabbix_trap_receiver.pl`**.
4. Lo script formatta i dati e li appende a un **file di testo** (es. `/var/log/zabbix/zabbix_traps.tmp`).
5. **Zabbix Server** (processo *SNMP trapper*) monitora costantemente questo file, legge le nuove trap e le associa agli Host corretti nel database tramite l'IP di origine.

---

## 🛠️ Prerequisiti
* Server con Zabbix 7.0 installato e funzionante.
* Accesso root o privilegi `sudo`.
* Connettività di rete dai dispositivi al server Zabbix sulla porta **162 UDP**.

---

## Passo 1: Installazione delle dipendenze

Dobbiamo installare il demone `snmptrapd`, gli strumenti SNMP e i moduli Perl necessari.
Si consiglia di installare anche i pacchetti dei MIB per permettere al server di tradurre gli OID numerici in nomi testuali comprensibili.

**Ubuntu / Debian:**
```bash
sudo apt update
sudo apt install snmp snmpd snmptrapd libsnmp-perl snmp-mibs-downloader
# Abilita l'uso dei MIB testuali commentando la riga "mibs :"
sudo sed -i 's/^mibs :/#mibs :/g' /etc/snmp/snmp.conf
```

**RHEL / Rocky Linux / AlmaLinux:**
```bash
sudo dnf install net-snmp net-snmp-utils net-snmp-perl
```

---

## Passo 2: Recuperare lo Script Perl di Zabbix

Lo script ufficiale è `zabbix_trap_receiver.pl`. Zabbix lo aggiorna periodicamente e il link diretto potrebbe cambiare. 

### Opzione A: Come trovare lo script manualmente (Metodo raccomandato)
Se il link diretto non dovesse più funzionare nel tempo, ecco come trovare la versione corretta:
1. Vai sul repository Git ufficiale: [https://git.zabbix.com](https://git.zabbix.com)
2. Naviga in **Zabbix** -> **Zabbix** (il progetto principale).
3. Nel menu a tendina in alto a sinistra (Branch/Tag), seleziona la tua versione (es. `release/7.0`).
4. Naviga nelle cartelle: `misc` -> `snmptrap`.
5. Clicca sul file `zabbix_trap_receiver.pl`.
6. Clicca sul pulsante **"Raw"** (in alto a destra nel riquadro del codice) e copia l'URL della pagina.

### Opzione B: Download diretto per Zabbix 7.0
Usa questo comando per scaricarlo direttamente in `/usr/bin/`:
```bash
sudo wget https://git.zabbix.com/projects/ZBX/repos/zabbix/raw/misc/snmptrap/zabbix_trap_receiver.pl?at=refs%2Fheads%2Frelease%2F7.0 -O /usr/bin/zabbix_trap_receiver.pl

# Rendilo eseguibile
sudo chmod +x /usr/bin/zabbix_trap_receiver.pl
```

---

## Passo 3: Configurazione del file temporaneo (Best Practice)

Di default, lo script Perl scrive le trap in `/tmp/zabbix_traps.tmp`. In ambienti di produzione **NON** è consigliato usare `/tmp/` perché il file viene cancellato ai riavvii e potrebbe causare problemi con SELinux. Utilizzeremo `/var/log/zabbix/`.

1. Crea il file e imposta i permessi corretti:
   ```bash
   sudo touch /var/log/zabbix/zabbix_traps.tmp
   sudo chown zabbix:zabbix /var/log/zabbix/zabbix_traps.tmp
   ```

2. Modifica lo script Perl per puntare al nuovo file:
   ```bash
   sudo nano /usr/bin/zabbix_trap_receiver.pl
   ```
   Trova la variabile `$SNMPTrapperFile` (di solito nelle primissime righe) e modificala così:
   ```perl
   $SNMPTrapperFile = '/var/log/zabbix/zabbix_traps.tmp';
   ```

---

## Passo 4: Configurazione di `snmptrapd`

Configuriamo il demone per accettare le trap e passarle allo script.

1. Apri il file di configurazione:
   ```bash
   sudo nano /etc/snmp/snmptrapd.conf
   ```

2. Sostituisci o aggiungi il seguente contenuto:
   ```text
   # --- ESEMPIO SNMPv2c ---
   # Sostituisci "public" con la tua vera stringa di community
   authCommunity log,execute,net public

   # --- ESEMPIO SNMPv3 (Opzionale) ---
   # createUser -e 0x0102030405 myuser SHA mypassword AES myprivpassword
   # authUser log,execute,net myuser

   # Formattazione per Zabbix (non rimuovere)
   perl do "/usr/bin/zabbix_trap_receiver.pl";
   ```

---

## Passo 5: Configurazione di Zabbix Server

Dobbiamo istruire Zabbix affinché legga il file che abbiamo appena configurato.

1. Apri il file del server:
   ```bash
   sudo nano /etc/zabbix/zabbix_server.conf
   ```

2. Trova le seguenti variabili, decommentale e impostale in questo modo:
   ```ini
   StartSNMPTrapper=1
   SNMPTrapperFile=/var/log/zabbix/zabbix_traps.tmp
   ```

3. Riavvia tutti i servizi e abilita l'avvio automatico:
   ```bash
   sudo systemctl restart snmptrapd zabbix-server
   sudo systemctl enable snmptrapd
   ```

*(Ricorda di aprire la porta UDP 162 sul firewall del tuo server: `sudo ufw allow 162/udp` oppure `sudo firewall-cmd --add-port=162/udp --permanent && sudo firewall-cmd --reload`)*.

---

## Passo 6: Configurazione nel Frontend Zabbix

Affinché la trap non venga scartata come "Unmatched", l'IP sorgente della trap deve corrispondere all'IP dell'interfaccia SNMP configurata sull'Host in Zabbix.

1. Vai su **Data collection** -> **Hosts**.
2. Apri (o crea) il tuo Host. Nella sezione **Interfaces**, assicurati che ci sia un'interfaccia di tipo **SNMP** con l'indirizzo IP identico a quello del dispositivo mittente.
3. Vai negli **Items** dell'Host e clicca su **Create item**:
   * **Name**: `SNMP Traps Fallback` (o simili)
   * **Type**: `SNMP trap`
   * **Key**: `snmptrap.fallback` *(cattura tutte le trap di questo IP. Se vuoi catturare solo trap specifiche usa le regex, es: `snmptrap["linkDown"]`)*
   * **Type of information**: `Log`
   * **Log time format**: `hh:mm:ss yyyy/MM/dd` *(utile per il parsing esatto della data)*
4. Salva.

---

## 🧪 Metodi di Test

Per essere sicuri che la catena funzioni, ecco 3 step di test per isolare eventuali problemi.

### Test 1: Test Locale (Generazione fittizia)
Dal terminale del server Zabbix, lancia una trap verso localhost:
```bash
snmptrap -v 2c -c public 127.0.0.1 "" 1.3.6.1.4.1.1.1.1 1.3.6.1.4.1.1.1.1 s "TEST_ZABBIX_LOCAL_TRAP"
```
* **Verifica**: Controlla il file con `tail -n 15 /var/log/zabbix/zabbix_traps.tmp`. Dovresti vedere i dati grezzi contenenti la stringa "TEST_ZABBIX_LOCAL_TRAP". 
* *Se non lo vedi, il problema è tra `snmptrapd` e lo script Perl.*

### Test 2: Verifica della ricezione di rete (`tcpdump`)
Se la trap parte da un router remoto ma non arriva, controlla se il traffico raggiunge il server:
```bash
sudo apt install tcpdump  # o dnf install tcpdump
sudo tcpdump -i any udp port 162
```
Forza il tuo apparato di rete a inviare una trap. Sul terminale dovresti vedere i pacchetti in arrivo. 
* *Se non li vedi, il problema è il firewall di rete o del server.*

### Test 3: Verifica sul Frontend Zabbix
Se Test 1 e Test 2 funzionano:
1. Vai su **Monitoring** -> **Latest data** nel frontend di Zabbix.
2. Filtra per il tuo Host.
3. Controlla l'Item `SNMP Traps Fallback`.
4. Se vedi i dati, la configurazione è **perfetta**. 

---

## 🚑 Troubleshooting Avanzato

* **Vedo le trap nel file di testo, ma non su Zabbix (Latest Data):**
  Guarda il log del server Zabbix:
  ```bash
  tail -f /var/log/zabbix/zabbix_server.log
  ```
  Se vedi il messaggio `unmatched trap received from[192.168.x.x]`, significa che Zabbix non sa a chi assegnare la trap. Assicurati che l'IP 192.168.x.x sia configurato come interfaccia SNMP in uno degli Host.

* **Nessun file viene creato o permission denied:**
  Se usi RedHat/AlmaLinux/Rocky, **SELinux** bloccherà di default `snmptrapd` dal creare/scrivere file e dall'eseguire script Perl.
  *Per testare se è SELinux:* esegui `sudo setenforce 0` e lancia il *Test 1*. Se funziona, devi creare un'eccezione SELinux:
  ```bash
  # Ripristina SELinux
  sudo setenforce 1
  # Dai il contesto corretto al file log e allo script
  sudo semanage fcontext -a -t zabbix_log_t "/var/log/zabbix(/.*)?"
  sudo restorecon -Rv /var/log/zabbix/
  # Permetti a snmptrapd di eseguire operazioni specifiche
  sudo setsebool -P zabbix_can_network 1
  ```
  *(Nota: Se SELinux continua a bloccare il demone, controlla i log di audit con `sudo ausearch -m avc -ts recent` e usa `audit2allow` per creare la policy).*

* **Formattazione delle trap incomprensibile (solo numeri OID):**
  Se al posto di `linkDown` vedi solo numeri come `1.3.6.1.4.1.x.x...`, manca il MIB corrispondente. Installa il pacchetto MIB (`snmp-mibs-downloader`) o posiziona i file MIB dei tuoi vendor nella cartella `/usr/share/snmp/mibs/` e riavvia `snmptrapd`.

---
*Ti è stata utile questa guida? Lascia una ⭐ al repository!*
