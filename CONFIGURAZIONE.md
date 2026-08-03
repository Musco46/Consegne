# Configurazione dell'accesso

Il sito è pubblico (GitHub Pages) e la chiave Supabase che sta in `index.html` è **pubblicabile
per progetto**: non è un segreto e non protegge nulla. La protezione sono le policy della
tabella, che devono rispondere solo a chi è autenticato.

Finché i passaggi qui sotto non sono fatti, il sito mostra la schermata del codice e non
lascia entrare nessuno.

## 1. I due utenti

Authentication → Users → **Add user**, uno per riga:

| Email | A cosa serve | Durata dell'accesso |
|---|---|---|
| `reparto@consegne.ria` | Computer condivisi del reparto | 12 ore, poi richiede il codice |
| `personale@consegne.ria` | Telefono o tablet personale | Non scade |

La **password è il codice** che si digita nella schermata di accesso. Vanno diverse fra loro.
Le email non sono segrete (sono in `index.html`): sono solo identificativi, non caselle vere,
quindi va spuntato *Auto Confirm User*.

I due utenti hanno **gli stessi identici permessi**: vedono e possono eliminare tutte le
consegne. Cambia solo per quanto tempo la sessione resta aperta sul dispositivo.

## 2. Chiudere le registrazioni

Authentication → Sign In / Providers → **Allow new users to sign up: OFF**.

È il passaggio che si dimentica più facilmente ed è quello che conta di più: se resta attivo,
chiunque si crea un account da solo, diventa "autenticato" e le policy qui sotto lo fanno
entrare. Senza questo, tutto il resto non serve a niente.

## 3. Le policy della tabella

Prima si guarda cosa c'è adesso:

```sql
select policyname, cmd, roles, qual, with_check
from pg_policies
where schemaname = 'public' and tablename = 'consegne_turno';
```

Vanno eliminate tutte quelle che nominano `anon` o `public`, e sostituite con queste quattro
(tutto il personale vede e può eliminare tutto — è voluto):

```sql
alter table public.consegne_turno enable row level security;

create policy "personale: lettura" on public.consegne_turno
  for select to authenticated using (true);
create policy "personale: inserimento" on public.consegne_turno
  for insert to authenticated with check (true);
create policy "personale: modifica" on public.consegne_turno
  for update to authenticated using (true) with check (true);
create policy "personale: eliminazione" on public.consegne_turno
  for delete to authenticated using (true);
```

Per verificare che sia chiusa davvero, da un terminale qualsiasi:

```bash
curl "https://kxqrtdlbqyhqaumvlafu.supabase.co/rest/v1/consegne_turno?select=*" \
  -H "apikey: sb_publishable_3Twp4s3IEtLu5EOtJ4bW-g_J4xoSErd"
```

Deve rispondere `[]` (o un errore di permessi). Se restituisce delle consegne, è ancora aperta.

## 4. Facoltativo, ma consigliato

Authentication → Sessions → **Time-box user sessions: 12 ore**, se il piano lo prevede.

Le 12 ore del codice di reparto sono controllate dal browser: basta a chiudere il turno su una
postazione, ma chi avesse in mano il dispositivo e sapesse cosa fare potrebbe allungarle.
Impostandolo anche qui, a farle rispettare è il server. Attenzione: vale per **tutti** gli
utenti, quindi scadrebbe anche il codice personale.

## Note

- I dati già presenti prima di questa configurazione vanno considerati **potenzialmente già
  letti**: la tabella è stata raggiungibile senza autenticazione da un repository pubblico.
- Il codice condiviso non dà **tracciabilità individuale**: non si può sapere chi ha cancellato
  cosa. È una scelta consapevole, in cambio di non dover fare il login a ogni cambio turno.
- Per cambiare un codice basta cambiare la password dell'utente su Supabase. Al successivo
  accesso i dispositivi lo richiederanno.
