# Configurazione dell'accesso

Il sito è pubblico (Vercel, `consegneria.vercel.app`) e la chiave Supabase che sta in
`index.html` è **pubblicabile per progetto**: non è un segreto e non protegge nulla. La
protezione sono le policy della tabella.

> **Il progetto Supabase è condiviso con un altro sito.** In Supabase la lista utenti è unica
> per progetto: gli account dell'altro sito sono autenticati anche qui. Per questo le policy
> qui sotto non dicono *"chi ha fatto l'accesso"* ma **"chi è uno di questi due account"** —
> altrimenti gli utenti dell'altro sito leggerebbero le consegne.

Finché i due passaggi non sono fatti, il sito mostra la schermata del codice e non lascia
entrare nessuno.

## 1. I due utenti

Authentication → Users → **Add user** → *Create new user*. Due volte:

| Email | A cosa serve | Durata dell'accesso |
|---|---|---|
| `reparto@consegne.ria` | Computer condivisi del reparto | 12 ore, poi richiede il codice |
| `personale@consegne.ria` | Telefono o tablet personale | Non scade |

La **password è il codice** che si digita nella schermata di accesso. Vanno diverse fra loro.
Spuntare **Auto Confirm User**: le email non sono caselle vere, sono solo etichette, e il
suffisso `@consegne.ria` serve a riconoscerle a colpo d'occhio in mezzo agli utenti dell'altro
sito.

I due account hanno **gli stessi identici permessi**: vedono e possono eliminare tutte le
consegne. Cambia solo per quanto tempo la sessione resta aperta sul dispositivo — e quello lo
decide l'app, non Supabase.

**Crearli subito.** Finché non esistono, quegli indirizzi sono liberi: sono scritti in
`index.html`, che è pubblico, e chi li registrasse per primo si troverebbe in mano l'accesso.

## 2. Le policy della tabella

SQL Editor → New query → incollare **tutto questo** → Run. Toglie da solo le policy esistenti,
qualunque nome abbiano, e mette le nuove.

```sql
alter table public.consegne_turno enable row level security;

-- Chi è "personale del reparto". Sta in un punto solo: per aggiungere un terzo codice si
-- crea l'utente e si aggiunge qui l'indirizzo, senza toccare le quattro policy.
-- Legge l'email dal token, non dalla tabella utenti: non è falsificabile lato client.
create or replace function public.personale_consegne()
returns boolean language sql stable as $$
  select (auth.jwt() ->> 'email') in (
    'reparto@consegne.ria',
    'personale@consegne.ria'
  )
$$;

-- via le policy attuali, che rispondono anche a chi non ha fatto l'accesso
do $$
declare p record;
begin
  for p in select policyname from pg_policies
           where schemaname = 'public' and tablename = 'consegne_turno'
  loop
    execute format('drop policy %I on public.consegne_turno', p.policyname);
  end loop;
end $$;

create policy "consegne: lettura" on public.consegne_turno
  for select to authenticated using (public.personale_consegne());
create policy "consegne: inserimento" on public.consegne_turno
  for insert to authenticated with check (public.personale_consegne());
create policy "consegne: modifica" on public.consegne_turno
  for update to authenticated using (public.personale_consegne())
                              with check (public.personale_consegne());
create policy "consegne: eliminazione" on public.consegne_turno
  for delete to authenticated using (public.personale_consegne());
```

### Verifica

Da un terminale qualsiasi, senza aver fatto alcun accesso:

```bash
curl "https://kxqrtdlbqyhqaumvlafu.supabase.co/rest/v1/consegne_turno?select=*" \
  -H "apikey: sb_publishable_3Twp4s3IEtLu5EOtJ4bW-g_J4xoSErd"
```

Deve rispondere `[]` (o un errore di permessi). Se elenca delle consegne, è ancora aperta.

## Cosa NON fare

**Non spegnere le registrazioni pubbliche** (Authentication → Sign In / Providers). È un
interruttore che vale per tutto il progetto: spegnerlo bloccherebbe le iscrizioni dell'altro
sito. Con le policy legate ai due account non serve — chi si registrasse per conto suo non
sarebbe nessuno dei due indirizzi e sulle consegne non vedrebbe niente.

## Resta aperto: le tabelle dell'altro sito

I due account del reparto sono utenti autenticati dell'**intero progetto**. Se le tabelle
dell'altro sito hanno policy del tipo `to authenticated using (true)`, chiunque abbia il codice
di reparto — che vivrà su un post-it accanto a un computer — può leggerle.

Per controllare:

```sql
select tablename, policyname, cmd, roles, qual
from pg_policies
where schemaname = 'public' and tablename <> 'consegne_turno';
```

Se qualcuna dice `{authenticated}` con `qual` uguale a `true`, va ristretta anche lei ai suoi
utenti, con lo stesso metodo usato qui.

## Le consegne si cancellano da sole dopo 16 giorni

Sono dati clinici di bambini e non devono restare in archivio per sempre. La copia che conta
è quella che si incolla nella cartella: questa serve a riaprire il letto nei giorni vicini.

**La pulizia la fa l'app, non il database, e non è una scorciatoia.** Un `pg_cron` (o una
Edge Function programmata) andrebbe acceso a livello di **progetto**, e il progetto è
condiviso con l'altro sito: sarebbe una modifica anche a casa loro, per un problema che è
solo nostro. Dall'app invece non si tocca niente di condiviso — nessuna estensione, nessuna
funzione, nessuna chiave in più. Si usa la policy di eliminazione che questi due account
hanno già.

Come funziona, in breve:

- all'apertura dell'app, dopo l'accesso, parte una `delete` con il filtro
  `creato_il < adesso - 16 giorni`. È il database a scegliere le righe, l'app dice solo fin dove;
- `creato_il` è la data dell'**ultimo salvataggio**, quindi una consegna che si continua ad
  aggiornare non scade mai finché la si usa;
- l'ora viene chiesta al server (header `Date` della risposta REST), **mai** all'orologio del
  dispositivo: un computer con la data avanti cancellerebbe le consegne di oggi. Se il server
  non risponde con un'ora, non si cancella niente.

Il prezzo è che la pulizia avviene solo quando qualcuno apre l'app. In un reparto vuol dire
ogni turno, che per una scadenza di sedici giorni basta e avanza.

Per cambiare la durata si tocca `CONSERVAZIONE_MS` in `index.html`, e nient'altro.

## La libreria Supabase sta nel repo

`vendor/supabase-js-2.112.0.umd.js` è la libreria client, copiata qui invece di essere
scaricata da un CDN a ogni apertura. È il file che fa l'accesso e parla con la tabella:
vede il codice mentre lo si digita, il token di sessione e ogni consegna che passa.
Chiederlo a `cdn.jsdelivr.net` come `@2` ("l'ultima 2.x") voleva dire eseguire ogni giorno
codice che nessuno ha guardato, e restare senza archivio ogni volta che la rete
dell'ospedale filtra quel dominio.

**Non ha una scadenza.** Una 2.x ferma continua a funzionare: l'API REST di Supabase è
stabile dentro la major. Si sostituisce solo se c'è un motivo — un avviso di sicurezza sulla
libreria, o Supabase che dismette qualcosa che questa versione usa. Non serve ricordarsene.

Quando quel motivo arriva, da PowerShell (`$v` è la versione nuova):

```powershell
$v = "2.113.0"
$meta = Invoke-RestMethod "https://registry.npmjs.org/@supabase/supabase-js/$v"
Invoke-WebRequest $meta.dist.tarball -OutFile "$env:TEMP\sb.tgz"

# 1. il pacchetto è quello che npm dice che è
$h = [Convert]::ToBase64String(
  [Security.Cryptography.SHA512]::Create().ComputeHash(
    [IO.File]::ReadAllBytes("$env:TEMP\sb.tgz")))
if ("sha512-$h" -ne $meta.dist.integrity) { throw "hash diverso: fermarsi qui" }

# 2. si prende il file dal pacchetto, non dal CDN
tar -xzf "$env:TEMP\sb.tgz" -C "$env:TEMP"
Copy-Item "$env:TEMP\package\dist\umd\supabase.js" ".\vendor\supabase-js-$v.umd.js"
```

Poi si aggiorna l'unico `<script src="vendor/...">` in `index.html` con il nome nuovo, si
cancella il file vecchio e si prova a entrare: se la libreria non si carica l'app lo dice da
sé ("Archivio non raggiungibile") invece di rompersi in silenzio.

Il passaggio 1 non è cerimonia: è l'unico punto in cui si verifica che il file sia davvero
quello pubblicato. E va preso `dist/umd/supabase.js` — **non** `supabase.min.js`: quel nome
nel pacchetto npm non esiste, era jsDelivr a fabbricarlo al volo minificando l'altro. Il
file pubblicato è già minificato, pesa uguale.

## Note

- I dati già presenti vanno considerati **potenzialmente già letti**: la tabella è stata
  raggiungibile senza autenticazione, da un repository pubblico.
- Il codice condiviso non dà **tracciabilità individuale**: non si può sapere chi ha cancellato
  cosa. È una scelta consapevole, in cambio di non fare il login a ogni cambio turno.
- Per cambiare un codice basta cambiare la password dell'utente su Supabase. Al successivo
  accesso i dispositivi lo richiederanno.
