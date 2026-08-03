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

## Note

- I dati già presenti vanno considerati **potenzialmente già letti**: la tabella è stata
  raggiungibile senza autenticazione, da un repository pubblico.
- Il codice condiviso non dà **tracciabilità individuale**: non si può sapere chi ha cancellato
  cosa. È una scelta consapevole, in cambio di non fare il login a ogni cambio turno.
- Per cambiare un codice basta cambiare la password dell'utente su Supabase. Al successivo
  accesso i dispositivi lo richiederanno.
