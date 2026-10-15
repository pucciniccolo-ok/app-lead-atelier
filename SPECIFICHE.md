# App Lead – Laura Romagnoli Atelier
Specifiche per Claude Code. Leggi tutto il file prima di scrivere codice.

## 1. Cos'è
Web app installabile (PWA) per gestire i lead dell'atelier Laura Romagnoli Atelier (Via Jacopo Carrucci 40, Empoli).
La usa una commessa con poca dimestichezza con la tecnologia, dal telefono aziendale. Il proprietario (Niccolò) ha un account admin.
NON è un CRM: niente email, niente invii automatici, niente notifiche push.

Tutta l'interfaccia è in italiano.

## 2. Stack e vincoli tecnici
- Frontend: Vite + React + TypeScript, solo client (SPA).
- Backend: Supabase (Postgres + Auth + Edge Functions), progetto in UE (Frankfurt). Nessun altro servizio a pagamento.
- Hosting: Cloudflare Pages, collegato al repository GitHub. Dominio: lead.lauraromagnoliatelier.it.
- Compatibilità: Safari iOS 14+ e Chrome Android recenti. Impostare il build target a `safari14` / `es2019` e testare che non vengano usate API più recenti senza ripiego. Niente funzioni che richiedono iOS 16+.
- PWA: manifest (nome "Lead Atelier", icona, colori), installabile con "Aggiungi a Home" su iPhone e Android. Service worker solo per la cache dei file dell'app, NON per i dati e NON per le notifiche.
- Accessibile e grande: testo base 17px, pulsanti alti almeno 48px, contrasto AA.
- Colori: nero caldo #0E0C0A, crema #F7F2E8 / #F4EFE4, oro #B08D46 (per testo su chiaro usare #7A5C22). Font: Josefin Sans per i titoli, Nunito Sans per il resto (Google Fonts).
- Fuso orario: Europe/Rome per tutte le date e gli orari mostrati e calcolati.
- Il repository contiene anche questo file e un README con i comandi.

## 3. Utenti e permessi
- Registrazione pubblica disattivata. Gli utenti li crea Niccolò dal pannello Supabase.
- Tabella `profiles` (id = auth.users.id, nome, ruolo: 'admin' | 'commessa').
- Row Level Security attiva su TUTTE le tabelle. Solo utenti autenticati leggono e scrivono. Le funzioni admin (import, impostazioni, export, riepilogo, cestino definitivo) sono riservate a ruolo admin anche lato database, non solo nascoste nell'interfaccia.
- Sessione persistente: la commessa non deve rifare il login ogni giorno. Pulsante "Esci" nel menu.

## 4. Modello dati (proposta, migrations SQL nel repo)
`leads`
- id uuid, nome text (nome e cognome insieme), telefono text (formato E.164, es. +393401234567), email text, citta text, data_nozze date null, tipo ('sposo' | 'cerimonia_donna')
- stato ('da_chiamare' | 'non_risponde' | 'appuntamento' | 'non_interessato'), default 'da_chiamare'
- visto boolean default false (badge "NUOVO" finché la commessa non apre la scheda)
- importato_il timestamptz, import_id uuid null, origine ('csv' | 'sito' | 'manuale')
- appuntamento_inizio timestamptz null, appuntamento_durata_min int default 90
- promemoria_inviato_il timestamptz null
- esito ('venduto' | 'non_venduto') null, importo_vendita numeric null
- note text
- eliminato_il timestamptz null (cestino)
- creato_il, aggiornato_il

`imports`: id, nome_file, tipo, creato_il, creato_da, righe_lette, importati, saltati.

`templates`: id, nome, testo, ordine. Testi iniziali in sezione 9.

`attivita` (storico): id, lead_id null, utente_id, azione, dettagli jsonb, creato_il. Registrare: pressione su Chiama, apertura WhatsApp (con nome template), cambio stato, appuntamento creato/modificato, esito, spostamento nel cestino, ripristino, import.

`impostazioni`: una riga JSON con gli orari (sezione 7) e l'indirizzo dell'atelier.

## 5. Schermate (vedi mockup approvati)
Barra in basso per la commessa: Lead · Calendario · Messaggi. Per l'admin anche: Admin.

### 5.1 Lista lead
- Ricerca per nome o telefono.
- Filtri per stato (Tutti, Nuovi, Da chiamare, Non risponde, Appuntamento, Non interessato), per data di importazione (Oggi, Questa settimana, Tutti) e per tipo (Sposo, Cerimonia donna).
- Lead raggruppati per giorno di importazione, i più recenti in alto ("IMPORTATI OGGI · GIO 15 OTT").
- Ogni card: nome, città, data nozze (se c'è), badge NUOVO se non visto, chip stato, chip tipo, pulsanti grandi "Chiama" e "WhatsApp".
- "Chiama" apre `tel:` e registra l'azione. NON cambia lo stato (la commessa lo cambia a mano, spesso chiama dal fisso).
- "WhatsApp" apre la sezione Messaggi con il lead già selezionato.

### 5.2 Selezione multipla
- Pulsante "Seleziona": compaiono le caselle, "Seleziona tutti" (rispetta i filtri attivi).
- Azioni in basso: "Cambia stato" e "Elimina (N)".
- Eliminare = spostare nel cestino, con conferma: "Restano recuperabili per 30 giorni dal Cestino".

### 5.3 Scheda lead
- Dati: telefono, email, città, data nozze, data di importazione. Pulsante "Modifica dati" (tutti i campi modificabili, data nozze compresa).
- Stato: 4 pulsanti grandi.
- Appuntamento: giorno, ora, durata; "Modifica". Scegliere "Appuntamento" senza data apre la schermata appuntamento.
- Esito dopo l'appuntamento: Venduto / Non venduto; con Venduto compare il campo "Importo di vendita" in euro.
- Note libere (salvataggio automatico).
- Storico delle azioni sul lead.
- "Sposta nel cestino".

### 5.4 Fissa appuntamento
- Scelta del giorno: solo giorni aperti (domenica e giorni chiusi non selezionabili).
- Pulsanti con gli orari previsti per quel giorno (da impostazioni). Barrato = pieno; tratteggiato = un camerino ancora libero (solo nei giorni con 2 camerini).
- "Altro orario…" per un orario libero dentro l'orario di apertura.
- Avviso (non blocco) se: i camerini disponibili quel giorno sono già tutti occupati in quell'intervallo; oppure l'appuntamento finisce dopo la chiusura. Testo: "Entrambi i camerini sono occupati da X (9:30–11:00) e Y (10:00–11:30)" / "Alle 10:00–11:30 c'è già X". Pulsanti "Scegli un altro orario" e "Salva lo stesso".
- Salvando, lo stato diventa "appuntamento".

### 5.5 Calendario
- Striscia della settimana con un pallino nei giorni con appuntamenti; tocco su un giorno = elenco del giorno.
- In alto il riquadro "Promemoria da inviare oggi": lead con appuntamento nei prossimi 2 giorni e promemoria non ancora inviato. Pulsante "Invia" (apre Messaggi con template Promemoria e il lead selezionato). Quando inviato: "✓ Inviato".
- Elenco appuntamenti per giorno in ordine di ora: ora, nome, tipo, città. Tocco = scheda lead.
- Questa sezione sostituisce l'agenda cartacea: deve essere chiara e veloce.

### 5.6 Messaggi
- Passo 1: scegli il messaggio tra i template salvati.
- Passo 2: scegli uno o più contatti (ricerca per nome).
- Passo 3: per ogni contatto, anteprima del testo con i campi già compilati, modificabile; "Apri in WhatsApp" apre `https://wa.me/<numero senza +>?text=<testo codificato>`. Con più contatti si procede uno alla volta ("1 di 2"), tornando nell'app.
- Campi automatici nei template: {nome} (solo il primo nome), {giorno} (es. "sabato 25 ottobre"), {ora} (es. "11:00"), {indirizzo}. Se un campo non è disponibile (es. lead senza appuntamento), evidenziarlo e lasciare il testo modificabile.
- Inviando un template di tipo Promemoria, segnare `promemoria_inviato_il`.
- Gestione template (crea, modifica, elimina, ordina): accessibile sia alla commessa sia all'admin.
- Funziona con WhatsApp normale: nessuna API.

### 5.7 Admin – Importa
- Carica file .csv o .xlsx. Leggere anche il CSV originale di Meta (UTF-16, separato da tabulazioni): rilevare automaticamente codifica (UTF-8, UTF-16, Windows-1252) e separatore (virgola, punto e virgola, tabulazione).
- Mappatura colonne Meta: `full_name`→nome, `phone`→telefono (togliere il prefisso "p:"), `email`, `city`→citta, `created_time` ignorato. Accettare anche intestazioni italiane (nome, cognome uniti in nome; telefono; email; città; data di nozze).
- Normalizzare il telefono: togliere spazi, punti e trattini; se manca il prefisso aggiungere +39; scartare con avviso i numeri non validi.
- Scelta del tipo per tutto il file (Sposo / Cerimonia donna).
- Riepilogo prima di importare: righe lette, nuovi, già presenti. Per ogni già presente (stesso telefono, anche nel cestino) mostrare il lead esistente con il telefono evidenziato, la data di importazione e lo stato, e far scegliere "Salta" (default) o "Importa lo stesso".
- Un import non cancella né sovrascrive mai lead esistenti.
- Se il file non ha la data di nozze, mostrare: "Questo file non contiene la data di nozze: la commessa potrà aggiungerla nella scheda del lead".

### 5.8 Admin – Riepilogo
- Periodo: Settimana, Mese, Da sempre.
- Numeri: lead importati, contattati (almeno un'azione registrata), appuntamenti, venduti con somma degli importi. Più "da lavorare": nuovi non aperti e in "Non risponde".
- Attività della commessa: elenco cronologico dallo storico, filtrabile per giorno.
- Strumenti: Importa, Esporta Excel (tutti i lead con tutti i campi, .xlsx), Cestino (ripristina o elimina per sempre, anche in blocco), Messaggi salvati, Impostazioni.

### 5.9 Admin – Impostazioni
- Per ogni giorno della settimana: aperto/chiuso, fasce di apertura, numero di camerini, orari degli appuntamenti previsti.
- Durata predefinita dell'appuntamento.
- Indirizzo dell'atelier (usato nei messaggi).

## 6. Cestino
- Eliminazione singola e multipla = cestino. Ripristino possibile.
- Dopo 30 giorni cancellazione definitiva: eseguita da una funzione del database chiamata all'apertura dell'app (niente servizi a pagamento o cron obbligatori).

## 7. Orari iniziali (modificabili dalle impostazioni)
Durata appuntamento: 90 minuti.
- Lunedì: aperto 16:00–20:00 · 1 camerino · appuntamenti 16:00, 17:30
- Martedì–venerdì: 10:00–13:00 e 16:00–20:00 · 1 camerino · appuntamenti 10:00, 11:30, 16:00, 17:30
- Sabato: 9:30–13:00 e 16:00–20:00 · 2 camerini · appuntamenti 9:30, 10:00, 11:00, 11:30, 16:00, 16:30, 17:30, 18:00
- Domenica: chiuso

## 8. Lead dal sito (fase finale)
- Supabase Edge Function `lead-sito` che riceve il Webhook del form Elementor Pro (azione "Webhook" dopo l'invio).
- Protetta con una chiave segreta nell'URL o nell'intestazione; rifiutare richieste senza chiave.
- Campi: nome, cognome, telefono, email, città, data di nozze. Tipo = sposo, origine = 'sito'. Stessa normalizzazione del telefono; se il telefono esiste già, creare comunque il lead ma segnarlo come possibile doppione visibile in lista.
- Documentare nel README l'URL da incollare in Elementor e i nomi dei campi (ID dei campi del form).

## 9. Template iniziali
- Primo contatto: "Ciao {nome}, sono di Laura Romagnoli Atelier di Empoli. Abbiamo ricevuto la tua richiesta per l'abito: quando ti farebbe comodo passare in atelier per vedere i modelli con calma? Ti aspettiamo in {indirizzo}."
- Non risponde: "Ciao {nome}, ti abbiamo cercato al telefono da Laura Romagnoli Atelier per la tua richiesta. Rispondi pure qui quando hai un momento e fissiamo insieme l'appuntamento."
- Conferma appuntamento: "Ciao {nome}, ti confermiamo l'appuntamento da Laura Romagnoli Atelier {giorno} alle {ora}, in {indirizzo}. A presto!"
- Promemoria: "Ciao {nome}! Ti ricordiamo l'appuntamento da Laura Romagnoli Atelier {giorno} alle {ora}, in {indirizzo}. Se hai un imprevisto rispondi pure a questo messaggio. A presto!"

## 10. Come lavorare
- Procedere per fasi; alla fine di ogni fase: build senza errori, test su viewport 390px e 360px, nessun errore in console, e un riepilogo per Niccolò in italiano semplice di cosa provare sul telefono.
- Fasi: (1) progetto, database, login, ruoli; (2) lista, scheda, stati, selezione multipla, cestino; (3) import CSV/Excel con doppioni; (4) appuntamenti, calendario, impostazioni orari; (5) messaggi e promemoria; (6) riepilogo admin, storico, export; (7) PWA e pubblicazione su Cloudflare Pages; (8) webhook del sito.
- Non inventare dati, nessun prezzo nell'app oltre all'importo di vendita inserito a mano.
- Mai mettere chiavi segrete nel codice del frontend: solo la chiave pubblica (anon/publishable) di Supabase; la sicurezza la fa RLS.
- Nessun backup automatico sul piano gratuito: l'export Excel serve anche da backup; ricordarlo nel README.
