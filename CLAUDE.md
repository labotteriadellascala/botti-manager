# CLAUDE.md — Botti Manager (gestionale)

Istruzioni di progetto per Claude Code. **Leggile a inizio di ogni sessione.**
Titolare: Andrea Canovi — Steel Head / La Botteria della Scala, P.IVA 04775640230.
Lo usano ogni giorno **Andrea** (dirige, non scrive codice — spesso da mobile) e **Giulia** (commerciale: preventivi, contenuti, contatti).
**Tutto — testi UI, commenti, comunicazioni — va scritto in italiano.**

---

## 0. La cosa più importante di tutte

Questo è un **gestionale in produzione, usato tutti i giorni, con dentro dati fiscali e contabili veri.** Non è un sito vetrina. Un errore qui non è una foto storta: è una fattura sbagliata, un ordine perso, una timbratura falsata, o Giulia che si ritrova il lavoro sovrascritto.

> **Regole d'oro (non negoziabili):**
> 1. **Mai `git push` senza l'OK esplicito di Andrea.** Prima mostra il diff, spiega cosa cambia e cosa potrebbe rompersi, aspetta conferma.
> 2. **Una decisione alla volta.** Scomponi le richieste complesse, conferma la direzione prima di implementare.
> 3. **Bastian contrario per contratto:** solleva obiezioni e tradeoff PRIMA di eseguire, con ragionamento concreto. Andrea decide in fretta se le opzioni sono chiare.
> 4. **Niente default silenziosi.** Se prendi una decisione di design al posto suo, diglielo. Se un valore è ambiguo, chiedi.

---

## 1. Deploy, versione, concorrenza con Giulia

- Repo: `labotteriadellascala/botti-manager` (**privato**), branch `main`.
- Online su **https://gestionale.labotteriadellascala.it** (GitHub Pages, deploy automatico a ogni push).
- **`git pull` a inizio sessione:** Andrea a volte modifica ancora dal web di GitHub, quindi la copia locale può essere vecchia. Lavorare su sorgente stale è la causa n.1 di regressioni.
- **Giulia lavora sul gestionale in contemporanea.** Dopo ogni deploy, Giulia deve fare **hard refresh (Ctrl+Shift+R)**, altrimenti vede la versione vecchia. Ricordalo ad Andrea quando pubblicate.
- **Operazioni distruttive/bulk** (es. reset giacenze magazzino): vanno fatte quando **Giulia NON sta editando**, per non collidere.
- **Stringa versione + `version.json` + `APP_VERSION`: TRE punti da tenere SEMPRE allineati** a ogni deploy (introdotti in v2.27 per l'avviso "nuova versione", vedi §10):
  1. La **stringa versione** in cima alla sidebar, riga tipo `v2.27 — 08/08/2026 · [changelog]` (attualmente riga ~548): nuovo numero, data e una riga di changelog.
  2. La costante **`window.APP_VERSION`** nel codice (vicino al collegamento Supabase): stesso numero, senza la `v` (es. `'2.27'`).
  3. Il file **`version.json`** nella radice del repo: `{"version":"2.27"}`, stesso numero.
  Se i tre non coincidono, l'avviso di aggiornamento non funziona (o parte a sproposito). **Un solo bump di versione = un solo commit per sessione.**

---

## 2. Architettura

**Un unico file `index.html`** (~9.800 righe), CSS e JS inline, nessun build step, nessun framework. Unica dipendenza esterna: **Supabase JS v2** via CDN (jsdelivr). **SheetJS (`xlsx@0.18.5`)** caricato lazy solo per l'export Excel.

- Navigazione: funzione **`nav('<modulo>', this)`** che cambia vista. Ogni modulo ha una funzione `render<Nome>()`.
- Lo stato globale gira su **`window.*`** (usato ~235 volte). **I `const` tra blocchi `<script>` diversi NON sono affidabili** (6 blocchi separati): per condividere stato usa `window.*` o helper locali, mai riferimenti a `const` di un altro blocco.

---

## 3. I moduli (voci di menu)

`dashboard` · `ordini` · `ordini-evento` · `contatti` · `lead` (marketing) · `nuovo-ordine` · `produzione` · `pianificazione` (planner settimanale) · `materiali` · `prodotti` · `accessori-listino` · `impostazioni` · `calcolo-prezzi` · `promemoria` · `finanze` (prima nota) · `mastrini`.
Più: **timbrature** (chiosco con **PIN**, `renderPinGate`) e **fatturazione** (FatturaPA).

Promemoria (v2.26): lista multi-riga per ordine salvata in campo **JSONB `dati`**; "Fatto" è **non distruttivo** (flag + timestamp, resta barrato nello storico), "Riapri" lo riattiva. La vista centrale e il badge contano solo gli attivi.

---

## 4. Supabase (accesso ai dati)

- Progetto: `wdlfzdjzynlnkqjhectg.supabase.co`. Nel file c'è la **anon key** (`SUPABASE_KEY`): è **pubblica per progetto, va bene così**. La sicurezza è nelle **RLS + RPC security-definer**. **Mai** incollare la `service_role` key nel client; **mai** aprire permessi di scrittura diretta sulle tabelle.
- **Pattern di accesso corretto:** anon key + **RPC strette** (security-definer). Le scritture sensibili passano da RPC, non da `insert/update` diretti su tabella aperta.
- **Claude NON può toccare Supabase direttamente.** Quando serve una modifica al DB: **fornisci lo SQL**, e **Andrea lo esegue a mano** nello SQL Editor di Supabase.

Trappole Supabase imparate a caro prezzo:
- **Query separate per tabella, MAI union.** Lo SQL Editor tronca a 100 righe di default e una union ordinata alfabeticamente **scarta in silenzio le tabelle in fondo**. Una query per tabella.
- **`CREATE OR REPLACE FUNCTION` con parametri diversi NON sostituisce: crea un overload.** Prima fai **`DROP FUNCTION` con la firma esatta**, poi ricrea.
- ⚠️ **NON eseguire mai lo SQL della RPC `inserisci_lead_da_sito`.** L'implementazione in produzione è più sofisticata di qualsiasi bozza (rate limiting, dedup, classificazione UTM). Lasciarla stare.

### Modifiche che toccano SIA il codice SIA il database (compatibilità all'indietro)

Quando pubblichi, **il browser di Giulia NON si ricarica**: continua a girare la versione vecchia finché lei non fa hard refresh, e non sa nemmeno che esiste un aggiornamento. Se in quella finestra lei salva un ordine, parte la logica **vecchia** (es. `insert` secco nella tabella `ordini`, senza guardia). Perciò, quando una modifica richiede sia codice sia Supabase:

- **Cambia il database in modo compatibile all'indietro (expand/contract).** Aggiungi colonne **facoltative** (nullable, con default). **NON** rinominare/eliminare colonne e **NON** aggiungere vincoli `NOT NULL` nello stesso passaggio in cui esce il client nuovo: il client vecchio manderebbe la forma vecchia e il salvataggio fallirebbe o scriverebbe dati incompleti.
- **Ordine di rilascio:** prima la modifica DB (compatibile), poi il codice. Le rimozioni/rinomini ("contract") si fanno **in una sessione successiva**, quando tutti i client sono aggiornati.
- **Prima di pubblicare una modifica alla logica ordini/salvataggio, avvisa Giulia** (vive in un'altra città): "salva l'ordine aperto, non aprirne di nuovi per 5 minuti, poi Ctrl+Shift+R".
- Vale anche per le operazioni distruttive/bulk: farle quando Giulia non sta editando (vedi §1).

---

## 5. Concorrenza — locking ottimistico (NON romperlo)

Andrea e Giulia possono modificare lo stesso record insieme. La protezione:
- Ogni record tiene un **`updated_at`**; in memoria è **`o._updatedAt`**.
- Al salvataggio, la query ha **`.eq('updated_at', o._updatedAt)`**: se nel frattempo l'altra persona ha salvato, `updated_at` è cambiato e l'update **non passa** ("blocco netto"), invece di sovrascrivere alla cieca.
- Dopo un salvataggio riuscito, si **rimemorizza il nuovo `updated_at`**.
- In caso di conflitto: si **ricarica la verità dal cloud** e si avvisa l'utente di rifare la modifica.

**Regole:** non sostituire questo schema con "last-write-wins". Ogni nuova scrittura su record condivisi deve **rispettare `updated_at`**. Preferisci sempre **operazioni non distruttive** (flag + storico) alla cancellazione (come già fatto per promemoria, stati ordine, ricalcolo giacenze).

---

## 6. Fisco e legale (massima cautela — non decidere da solo)

Il modulo fatture genera **FatturaPA** verso lo **SDI**. Qui gli errori costano.
- Serie fatture: **`/BM`** (gestionale) è separata dalla serie **FPR** di Aruba. Non mescolarle.
- **`TD01`**, scorporo IVA **22% dal lordo**, `formatoTrasmissione`: logica fiscale delicata.
- **Non inviare/abilitare invii reali a SDI** finché non sono chiarite con il commercialista le tre domande aperte (trattamento **TD01 per acconti**, separazione serie **/BM**, **scorporo 22%** dal lordo). Prossimi step tecnici noti: validazione checksum CF/P.IVA e gestione dello scarto SDI.
- **Arrotondamenti nelle timbrature**: da validare col commercialista prima di applicarli.
- **Principio sentinella IVA (esplicito > silenzioso):** non auto-compilare mai valori che rendano "revisionato" indistinguibile da "dimenticato" (IVA sulle righe di banca, quantità materiali, default distinta base). Se un valore va rivisto da un umano, lascialo vuoto/segnalato, non riempirlo di nascosto.
- **Qualsiasi cosa tocchi fisco/legale va segnalata ad Andrea proattivamente**, non decisa di iniziativa.

---

## 7. Trappole tecniche ricorrenti (in questo file)

- **Collisioni di classi CSS**: file singolo enorme → nomi di classe generici causano bug visivi subdoli (già successo: `.bar` in conflitto, rinominato `.cmp-line`). Usa nomi specifici e cerca prima se un nome esiste già.
- **`backdrop-filter` intrappola lo stacking context su mobile** (già causato il drawer del menu bloccato). Usalo con estrema cautela; è presente in un punto.
- **`const` cross-block inaffidabili** → `window.*` o helper locali (vedi §2).
- **Mismatch `async`/`sync`**: categoria di bug ricorrente in questa architettura a file singolo. Attenzione alle funzioni che a volte sono await-ate e a volte no.

---

## 8. Metodo di lavoro

- Prima di consegnare/committare: **`node --check` su tutti i blocchi `<script>` inline** (sono ~6) + **mini unit test mock** mirati sulla modifica. È lo standard di qualità di questo progetto.
- Modifiche chirurgiche: cerca il punto giusto e cambia **solo quello**, non riscrivere interi moduli.
- Un commit per sessione, messaggio chiaro in italiano, e **aggiorna la stringa versione** (§1).
- Non introdurre build step o framework: deve restare un singolo file statico servito da GitHub Pages.

---

## 9. Cosa NON è in questo repo

- Il **sito vetrina** è un altro repo (`sito`, `www.labotteriadellascala.it`). Non c'entra.
- **Supabase, Meta, GA4**: servizi esterni, non file di questo repo.

---

## 10. Avviso "nuova versione disponibile" (v2.27)

Quando si pubblica una versione nuova, chi sta già usando il gestionale (soprattutto Giulia, da un'altra città) non se ne accorge: il browser continua a girare la versione vecchia finché non ricarica a mano. Per questo, dalla v2.27 c'è un avviso in-app.

- **Come funziona:** un blocco `<script>` isolato in fondo a `index.html` confronta `window.APP_VERSION` con la versione pubblicata in `version.json` (fetch `no-store` + cache-busting). Il controllo parte poco dopo l'avvio, al focus/ritorno sulla finestra, e ogni 5 minuti (con anti-rimbalzo di 60s).
- **Se online c'è una versione PIÙ ALTA** (confronto numerico per campi: `2.10 > 2.9`), compare una **fascetta discreta** in basso (`.bm-upd-*`) con pulsante **"Ricarica"**. Richiudibile, riappare al focus.
- **Non ricarica MAI da sola.** La ricarica la decide l'utente col pulsante — altrimenti si cancellerebbe un ordine in compilazione non salvato.
- **Confronto solo "più alta", non "diversa":** così un eventuale rollback a una versione più bassa non mostra l'avviso a chi ha già la più nuova.
- **Regola operativa:** a ogni deploy allinea i **tre punti** della versione (§1). Se dimentichi `version.json`, l'avviso non parte.
- **Limite noto:** il client già aperto prima di questo deploy non ha il controllo, quindi non vedrà la fascetta per il deploy che *introduce* la funzione. Funziona **dal deploy successivo in poi**.

---

## Reference rapida

**Tabelle:** `ordini`, `ordini_evento`, `prodotti`, `materiali`, `accessori`, `contatti`, `lead_marketing`, `dipendenti`, `timbrature`, `botti_fatture`, `allegati`, `impostazioni`, `dashboard_tv`, `tv_attivita`.

**RPC usate nel gestionale:** `apri_prodotto`, `chiudi_prodotto`, `annulla_apertura_prodotto`, `stato_prodotto`, `chiudi_ordine`, `timbra` (e, lato magazzino, `rileva_giacenza` / `aggiungi_giacenza`).

---

*Se una richiesta di Andrea contrasta con queste istruzioni, vince Andrea — ma faglielo notare prima di procedere. E su fisco, Supabase in produzione e operazioni distruttive: nel dubbio, fermati e chiedi.*
