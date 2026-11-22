# Postgresql HA mediante PITR in due VM-container

**DISCLAIMER:** sebbene **PostgreSQL** sia rilasciato sotto licenza open-source gratuita (*PostgreSQL License*), si precisa che la configurazione, gli script e l'intera infrastruttura descritti in questo repository sono stati realizzati **esclusivamente a fini di studio personale e laboratorio didattico**, senza alcuno scopo di lucro o uso in produzione; prima di adottare soluzioni simili in ambienti di produzione reali, è necessario adeguarle alle policy di sicurezza, reti e backup aziendali.

Laboratorio infrastrutturale su **Proxmox VE** dedicato alla protezione dei dati, alta affidabilità e disaster recovery su **PostgreSQL 18.7** (Debian 12 Bookworm).

## Obiettivo e Scelte Architetturali

In un ambiente di produzione, la sola replica, **Streaming Replication**, protegge dai guasti hardware ma replica istantaneamente anche gli errori umani (es. `DELETE` o `DROP` errati). Viceversa, il solo **PITR** su backup richiede tempi di ripristino più lunghi in caso di crash del server primario.

Questo laboratorio combina entrambi gli approcci:
1. **Business Continuity:** Nodo Standby in sola lettura sincronizzato in tempo reale via Streaming Replication.
2. **Point-In-Time Recovery (PITR):** Doppia archiviazione ridondata dei file WAL (locale + nodo remoto via `rsync`) per consentire il ripristino a un istante preciso nel passato.

## Topologia dell'Infrastruttura (Proxmox VE)

* **Nodo Primario (`192.168.1.153` - VM Debian 12):** Istanza `18/main` (porta `5432`). Esegue il backup fisico iniziale (`pg_basebackup`) e archivia ogni file WAL su due destinazioni distinte.
* **Nodo Standby & Archivio Remoto (`192.168.1.154` - Container LXC Debian 12):** Istanza secondaria inizializzata tramite `pg_basebackup -R` in modalità `hot_standby` e destinazione remota per i WAL archiviati.
* *(Nota storica di lab: una prima fase di test PITR ha previsto una seconda istanza `18/restore` sulla porta `5433` della VM primaria, pensata per risiedere su un disco dati dedicato prima della migrazione su nodo LXC separato).*

---

## Flusso Operativo Implementato

1. **Configurazione Primario & Doppia Archiviazione WAL:**
   * Impostazione `archive_mode = on` e `archive_command` con doppio salvataggio atomico (copia locale + trasferimento SSH/rsync verso il nodo LXC).
   * *(Nota: `wal_level = logical` è attivo per supportare contemporaneamente anche altre pipeline CDC (Debezium/Kafka verso MariaDB), mantenendo totale retrocompatibilità con il PITR).*
2. **Inizializzazione Standby Remoto:**
   * Clonazione online tramite `pg_basebackup -Fp -Xs -P -R` e verifica dello stato di recovery (`SELECT pg_is_in_recovery();`).
3. **Simulazione Disastro e Ripristino (`recovery.signal`):**
   * Arresto del primario, eliminazione simulata della `data_directory`, ripristino dal `base_full` e riapplicazione automatica dei WAL archiviati fino all'ultima transazione valida.
4. **Manutenzione Archivio WAL:**
   * Verifica dello stato tramite `pg_stat_archiver` e pulizia controllata dei log obsoleti con `pg_archivecleanup`.

---

## Problemi risolti e lezioni apprese:

* **Gestione `.pid` su cluster multipli Debian:** Disallineamento del parametro `external_pid_file` durante la clonazione locale delle istanze (`pg_lsclusters` segnala `down` un cluster attivo se il file `.pid` in `/var/run/postgresql/` viene rimosso da un'istanza gemella).
* **Container LXC Minimali (Debian 12):** Necessità di generare esplicitamente il locale `it_IT.UTF-8` per allinearsi al primario; installazione del pacchetto `dbus`, e commento della riga `session optional pam_systemd.so` nel file `common-session` per eliminare i timeout di autenticazione `pam_systemd` durante i trasferimenti `rsync` via SSH.

