Ecco come funziona il flusso:
**Dispositivo di rete** invia la trap -> **snmptrapd** (demone) la riceve sulla porta 162 UDP -> **snmptrapd** passa la trap allo **script Perl** -> Lo script formatta la trap e la scrive in un **file di testo** -> **Zabbix Server** legge il file e assegna la trap al corretto Host/Item.

Di seguito trovi la guida completa passo-passo per **Zabbix 7.0**.

---

### Passo 1: Installare i pacchetti necessari

Per prima cosa, devi installare il demone per le trap SNMP (`snmptrapd`) e il modulo Perl necessario per far comunicare il demone con lo script. Esegui i comandi in base al sistema operativo del tuo Zabbix Server.

**Per Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install snmp snmpd snmptrapd libsnmp-perl
```

**Per RHEL/Rocky Linux/AlmaLinux:**
```bash
sudo dnf install net-snmp net-snmp-utils net-snmp-perl
```

---

### Passo 2: Scaricare lo script Perl ufficiale di Zabbix 7.0

Scarica lo script direttamente dal repository Git ufficiale di Zabbix (ramo 7.0) e posizionalo in una cartella di sistema.

```bash
sudo wget https://git.zabbix.com/projects/ZBX/repos/zabbix/raw/misc/snmptrap/zabbix_trap_receiver.pl?at=refs%2Fheads%2Frelease%2F7.0 -O /usr/bin/zabbix_trap_receiver.pl
```

Rendi lo script eseguibile:
```bash
sudo chmod +x /usr/bin/zabbix_trap_receiver.pl
```

*(Nota opzionale: se apri il file con un editor di testo, vedrai che di default scriverà le trap nel file `/tmp/zabbix_traps.tmp`. Puoi lasciare questo valore di default o modificarlo, ma se lo modifichi dovrai usare lo stesso percorso al Passo 4).*

---

### Passo 3: Configurare `snmptrapd`

Devi istruire `snmptrapd` in modo che accetti le trap e le passi allo script Perl.

1. Modifica il file di configurazione di `snmptrapd`:
   ```bash
   sudo nano /etc/snmp/snmptrapd.conf
   ```

2. Aggiungi le seguenti righe alla fine del file:
   ```text
   # Sostituisci "public" con la tua community SNMP reale, se diversa
   authCommunity log,execute,net public
   
   # Carica lo script perl di zabbix
   perl do "/usr/bin/zabbix_trap_receiver.pl";
   ```
   *Nota: se usi SNMPv3, dovrai configurare l'utente v3 con la direttiva `createUser` e `authUser` invece di `authCommunity`.*

---

### Passo 4: Configurare Zabbix Server

Ora devi dire a Zabbix Server di abilitare i processi "Trapper" e di dirgli dove andare a leggere il file creato dallo script Perl.

1. Modifica il file di configurazione del server Zabbix:
   ```bash
   sudo nano /etc/zabbix/zabbix_server.conf
   ```

2. Trova le seguenti voci, decommentale (togliendo il `#`) e impostale così:
   ```text
   StartSNMPTrapper=1
   SNMPTrapperFile=/tmp/zabbix_traps.tmp
   ```

---

### Passo 5: Riavviare i servizi e configurare il Firewall

1. Riavvia i servizi per applicare le modifiche e abilita `snmptrapd` all'avvio:
   ```bash
   sudo systemctl restart snmptrapd zabbix-server
   sudo systemctl enable snmptrapd
   ```

2. **Firewall:** Assicurati che la porta **162 UDP** sia aperta sul server Zabbix per ricevere le trap dai dispositivi.
   - Su *ufw* (Ubuntu): `sudo ufw allow 162/udp`
   - Su *firewalld* (RHEL/Rocky): `sudo firewall-cmd --add-port=162/udp --permanent && sudo firewall-cmd --reload`

---

### Passo 6: Configurazione nel Frontend di Zabbix 7.0

Affinché Zabbix associ la trap al dispositivo corretto, l'indirizzo IP da cui arriva la trap deve corrispondere all'IP configurato nell'Host in Zabbix.

#### 6.1. Creare l'Host (o modificarne uno esistente)
1. Vai su **Data collection -> Hosts**.
2. Crea o apri l'Host di rete che ti invierà la trap.
3. Nella sezione **Interfaces**, assicurati che ci sia un'interfaccia **SNMP** configurata. **L'indirizzo IP deve essere esattamente quello con cui il dispositivo si presenta al server**. (La porta può rimanere 161, Zabbix capirà comunque l'associazione).

#### 6.2. Creare l'Item (Elemento) per raccogliere la Trap
Per leggere la trap, devi creare un Item di tipo "SNMP trap".

1. Vai negli **Items** di quell'Host e clicca su **Create item**.
2. Compila i campi in questo modo:
   - **Name**: Trap SNMP (Generico) o qualsiasi nome preferisci.
   - **Type**: `SNMP trap`
   - **Key**: `snmptrap.fallback` *(questa chiave cattura tutte le trap provenienti da questo IP non catturate da altre chiavi più specifiche)*.
   - **Type of information**: `Log`
   - **Log time format**: `hh:mm:ss yyyy/MM/dd` *(opzionale, aiuta Zabbix a parsare l'orario)*
3. Clicca su **Add**.

*Vuoi catturare una trap specifica anziché tutte?*
Se vuoi filtrare solo un certo tipo di trap, ad esempio quando un link va giù, puoi usare come **Key** una regex, per esempio: `snmptrap["linkDown"]` oppure `snmptrap["1.3.6.1.4.1.9.9.43"]`.

---

### Passo 7: Testare il funzionamento

Per testare se tutto funziona, puoi generare una trap finta dal tuo stesso server Zabbix (o da una macchina remota) e inviarla a `snmptrapd`.

Esegui questo comando dal terminale del tuo server Zabbix:
```bash
snmptrap -v 2c -c public 127.0.0.1 "" 1.3.6.1.4.1.1.1.1 1.3.6.1.4.1.1.1.1 s "Test trap per Zabbix"
```

**Come verificare che sia andata a buon fine:**
1. Controlla il file temporaneo:
   ```bash
   cat /tmp/zabbix_traps.tmp
   ```
   *Se vedi del testo formattato contenente "Test trap per Zabbix", significa che `snmptrapd` e lo script Perl stanno lavorando perfettamente!*

2. Vai nel frontend di Zabbix: **Monitoring -> Latest data**. Cerca l'Host e l'Item `snmptrap.fallback` che hai creato. Dovresti vedere il testo della trap appena inviata.

---

### ⚠️ Risoluzione dei problemi frequenti
* **Il file `/tmp/zabbix_traps.tmp` non viene creato**: Assicurati che in `snmptrapd.conf` ci sia la community corretta (`authCommunity ... public`).
* **SELinux (su RHEL/Rocky Linux)**: SELinux spesso blocca lo script Perl o la scrittura in `/tmp`. Per verificare se SELinux sta bloccando qualcosa, imposta temporaneamente `setenforce 0` e riprova. Se funziona, dovrai creare una policy SELinux ad-hoc o cambiare la directory del trap file in un percorso gradito a SELinux (es. `/var/lib/zabbix/snmptraps/`).
* **Traps in "Unmatched traps"**: Se le trap vengono scritte nel file `/tmp/zabbix_traps.tmp` ma su Zabbix non le vedi assegnate al tuo Host, vai a vedere il log di zabbix (`/var/log/zabbix/zabbix_server.log`). Probabilmente vedrai un messaggio `unmatched trap received from [192.168.x.x]`. Questo significa che l'IP `192.168.x.x` non è configurato su nessuna interfaccia SNMP in nessun Host Zabbix. Correggi l'IP sull'Host nel frontend.
