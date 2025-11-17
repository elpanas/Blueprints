# AutoCheckDeploy

Questo repository contiene uno **script di deploy automatizzato** per applicazioni containerizzate con Docker Compose. Lo script aggiorna l'applicazione su un server remoto via SSH, con backup automatici e possibilità di rollback in caso di errori.

---

## Funzionamento generale

Lo workflow è implementato come GitHub Action (`.github/workflows/deploy.yml`) e si attiva quando vengono effettuati push sui branch configurati (`main` o `dev` di default).  

Il processo di deploy segue questi step principali:

1. **Checkout del codice**  
   L'Action scarica i file di deploy dalla repository sul runner GitHub tramite `actions/checkout@v4`.

2. **Caricamento della configurazione**  
   - Legge il file `deploy.conf` presente nel repository.  
   - Esporta le variabili principali (`APP_NAME`, `APP_PATH`, `SERVICES`, `GIT_REPO`, `VOLUMES`, `HEALTHCHECK_TIMEOUT`) come environment variables per gli step successivi.

3. **Connessione al server via SSH**  
   - Utilizza l'action `appleboy/ssh-action@v0.1.10` per connettersi al server via SSH
   - Le credenziali sono prese dai GitHub Secrets: `SERVER_HOST`, `SERVER_USERNAME`, `SERVER_SSH_KEY`.

4. **Inizializzazione e preparazione**  
   - Definisce il branch da deployare (`main` o `dev` o altro nell'elenco branches).  
   - Converte le stringhe `SERVICES` e `VOLUMES` in liste.  
   - Crea cartelle di log e backup (`logs/` e `backups/`).  
   - Prepara un file di log con timestamp.

5. **Backup dei volumi**  
   - Ogni volume elencato in `VOLUMES` viene salvato in un file `.tar.gz` tramite un container Alpine temporaneo.  
   - I backup vengono scritti nella cartella `backups/` sul server.

6. **Backup delle immagini Docker per rollback**  
   - Ogni immagine dei servizi viene clonata come immagine di rollback (`<SERVICE>_rollback`).  
   - Permette di tornare alla versione precedente in caso di errori.

7. **Aggiornamento del repository**  
   - Se la cartella non contiene `.git`, clona la repo dal branch corretto.  
   - Se la cartella contiene già la repo, effettua `fetch` e `reset --hard` sul branch target per evitare conflitti.

8. **Stop dei container esistenti**  
   - Esegue `docker compose down` per fermare i container attivi e prepararsi al nuovo deploy

9. **Build e avvio dei nuovi container**  
   - Esegue `docker compose up --build -d`.  
   - Per ogni servizio:
     - Controlla il numero di riavvii e lo stato di salute (`healthcheck`).  
     - Se c’è un errore, esegue il **rollback automatico**.  
     - Mostra gli ultimi 50 log del container.

10. **Deploy completato**  
    - Conferma il deploy se tutti i servizi sono avviati correttamente.  
    - Le immagini di rollback rimangono conservate per eventuali future necessità.

---

## Variabili principali (`deploy.conf`)

| Variabile            | Descrizione |
|----------------------|-------------|
| `APP_NAME`           | Nome dell'applicazione (es. `your-app`) |
| `APP_PATH`           | Percorso sul server dove si trova il progetto (es. `/srv/your-app`) |
| `SERVICES`           | Lista dei servizi Docker Compose, separati da virgola (es. `backend,frontend`) |
| `GIT_REPO`           | URL del repository Git (es. `git@github.com:username/repo.git`) |
| `VOLUMES`            | Lista dei volumi Docker da backuppare, separati da virgola |
| `HEALTHCHECK_TIMEOUT`| Timeout in secondi per il controllo dello stato di salute dei container |

---

## Rollback automatico

Se un container fallisce l'healthcheck o si riavvia più volte, lo script:

1. Mostra gli ultimi log del container fallito.  
2. Salva i log completi in un file con timestamp.  
3. Ripristina l'immagine precedente.  
4. Riavvia i container con la versione funzionante.
5. Esce con codice 1, deploy fallito.
