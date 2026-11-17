# Crushly – Einrichtungsanleitung

Die App besteht aus drei Dateien und läuft komplett statisch (GitHub Pages) mit Supabase als Backend:

| Datei | Zweck |
| --- | --- |
| `index.html` | Die komplette Web-App (Tailwind CSS + Supabase SDK via CDN, Vanilla JS) |
| `supabase-setup.sql` | Tabellen, Trigger, RLS-Policies und Match-Funktion für Supabase |
| `ANLEITUNG.md` | Diese Anleitung |

---

## Teil 1: Supabase einrichten (ca. 10 Minuten)

### Schritt 1 – Projekt anlegen
1. Gehe zu [supabase.com](https://supabase.com) und melde dich an (GitHub-Login möglich).
2. Klicke auf **New project**, gib einen Namen ein (z. B. `crushly`), wähle als Region **Central EU (Frankfurt)** und setze ein Datenbank-Passwort (irgendwo notieren, wird später nicht mehr gebraucht).
3. Warte 1–2 Minuten, bis das Projekt bereit ist.

### Schritt 2 – SQL-Setup ausführen
1. Öffne im linken Menü den **SQL Editor** (Symbol `>_`).
2. Klicke **New query**, füge den **kompletten Inhalt** von `supabase-setup.sql` ein.
3. Klicke **Run**. Es muss unten `Success. No rows returned` erscheinen.

### Schritt 3 – E-Mail-Bestätigung deaktivieren (wichtig)
Die App benutzt KEINE echten E-Mail-Adressen (sie erzeugt intern Dummy-Adressen wie `max.mustermann@schule.local`). Deshalb muss die E-Mail-Bestätigung aus sein:

1. Öffne **Authentication → Sign In / Up** (in älteren Dashboards: *Authentication → Providers*).
2. Klicke beim **Email**-Provider auf die Einstellungen bzw. öffne **Emails**.
3. Schalte **„Confirm email“** aus.
4. Speichern mit **Save**.

> Solange „Confirm email“ an ist, kann sich niemand direkt nach der Registrierung einloggen – die App zeigt dann einen entsprechenden Hinweis.

### Schritt 4 – URL und Key in die App eintragen
1. Öffne **Project Settings** (Zahnrad) → **API**.
2. Kopiere **Project URL** und den **Publishable Key** (früher „anon public“-Key; neue Projekte zeigen den Key im Format `sb_publishable_…`).
3. Öffne `index.html` und trage beides ganz oben im Script-Bereich ein:

```js
const SUPABASE_URL = 'https://xxxxxxxx.supabase.co';
const SUPABASE_ANON_KEY = 'sb_publishable_…';
```

> ✅ Der Publishable Key ist dafür gemacht, öffentlich im Frontend zu liegen. Die Sicherheit kommt aus den RLS-Policies: Niemand kann damit fremde Crushes lesen.

---

## Teil 2: Auf GitHub Pages veröffentlichen (ca. 5 Minuten)

### Schritt 1 – Repository erstellen
1. Erstelle auf [github.com](https://github.com) ein **neues, öffentliches** Repository (z. B. `crushly`). *Ohne* README starten.
2. Lade die drei Dateien hoch (auf der leeren Repo-Seite: **uploading an existing file**) – `index.html`, `supabase-setup.sql`, `ANLEITUNG.md` – und committe mit **Commit changes**.
   (Oder per Kommandozeile: `git init && git add . && git commit -m "Crushly" && git remote add origin <repo-url> && git push -u origin main`)

### Schritt 2 – GitHub Pages aktivieren
1. In deinem Repo: **Settings** → links **Pages**.
2. Unter **Build and deployment** → **Source**: `Deploy from a branch`.
3. **Branch**: `main`, Ordner: `/ (root)` → **Save**.
4. Nach 1–2 Minuten ist die App live unter:
   **`https://<dein-benutzername>.github.io/crushly/`**

---

## Teil 3: Testen

1. Öffne die App-URL im Browser (Privatfenster) und registriere z. B. **Test Eins** (Passwort `test123`).
2. Öffne die URL in einem **zweiten** Privatfenster (anderer Browser geht auch) und registriere **Test Zwei**.
3. Speichere als Test Eins den Crush „Test Zwei“, als Test Zwei den Crush „Test Eins“.
4. Auf beiden Dashboards erscheint jetzt automatisch das **Match-Banner** – auch wenn sich der zweite Nutzer erst Tage später registriert.

Test-Accounts wieder entfernen: in der App unter **Account → Account endgültig löschen** (s. u.) oder über **Authentication → Users** im Dashboard; Profil und Crush-Einträge werden automatisch mitgelöscht (CASCADE).

---

## Teil 4: Account verwalten (Passwort & Löschen)

Im Dashboard gibt es unten den Bereich **Account** mit zwei Aktionen:

### Passwort ändern
1. Trage unter **Account → Passwort ändern** ein neues Passwort (mind. 6 Zeichen) ein und wiederhole es.
2. Klicke **Passwort ändern**. Die Änderung läuft über `supabase.auth.updateUser({ password })` direkt am angemeldeten Nutzer – kein erneutes Einloggen nötig. Beim nächsten Login ist das neue Passwort gültig.

### Account löschen
1. Klicke **Account endgültig löschen …**. Ein Bestätigungsdialog öffnet sich.
2. Tippe zur Bestätigung das Wort **LÖSCHEN** ein und klicke **Endgültig löschen**.
3. Die App ruft die serverseitige Funktion `delete_own_account()` auf. Diese löscht den eigenen `auth.users`-Eintrag; Profil und Crush-Auswahl werden per `ON DELETE CASCADE` automatisch mit entfernt. Anschließend wirst du abgemeldet.

> **Hinweis zu den Supabase-Regeln:** Supabase bietet **keine** clientseitige Methode, den eigenen `auth.users`-Eintrag zu löschen – `auth.admin.deleteUser()` benötigt den **service_role-Key**, der niemals im Frontend liegen darf. Daher ist das Löschen als `SECURITY DEFINER`-Funktion (`public.delete_own_account`) umgesetzt, die mit den Rechten von `postgres` läuft und **nur den aufrufenden, angemeldeten Nutzer** löscht. Dritte können die Funktion zwar aufrufen, löschen damit aber immer nur ihren eigenen Account.

---

## Wie die Sicherheit funktioniert

- **Row Level Security (RLS)** auf beiden Tabellen:
  - `profiles`: angemeldete Nutzer dürfen Namen **lesen** (für die Live-Suche) – mehr nicht.
  - `crushes`: jeder darf **nur seine eigenen** Einträge lesen/schreiben/ändern/löschen.
- **Match-Abgleich nur über die RPC-Funktion `get_my_match()`** (`SECURITY DEFINER`): Sie läuft serverseitig mit erweiterten Rechten, gibt aber **nur dann** ein Ergebnis zurück, wenn zwei Personen sich **gegenseitig** ausgewählt haben. Einseitige Crushes sind über die API nicht auslesbar.
- **Passwort ändern** läuft über `supabase.auth.updateUser({ password })` – eine offizielle Client-Methode, die ausschließlich den angemeldeten Nutzer betrifft. Der Service-Role-Key wird dafür nicht gebraucht.
- **Account löschen** über die RPC-Funktion `delete_own_account()` (`SECURITY DEFINER`, Owner `postgres`). Sie löscht **nur den aufrufenden** `auth.users`-Eintrag; Profil und Crushes fallen per `ON DELETE CASCADE` mit. Der Service-Role-Key bleibt im Backend – das Frontend braucht ihn nicht.
- **Anmeldung ohne E-Mail:** Aus Vor- und Nachname wird automatisch eine interne Dummy-Adresse (`vorname.nachname@schule.local`) erzeugt. Umlaute werden dabei e-mail-tauglich umschrieben (ü → ue …), Sonderzeichen entfernt.

## Tipps & Grenzen

- **Doppelte Namen** (zwei „Max Mustermann“ in einer Klasse) sind nicht möglich – der normalisierte Name muss einmalig sein. Betroffene Schüler können z. B. einen zweiten Vornamen oder eine Anfangsziffer verwenden („Max Mustermann 9b“).
- Das Match-Prüfung läuft alle 20 Sekunden automatisch im Hintergrund, sobald das Dashboard offen ist.
- Passwörter werden von Supabase gehasht gespeichert; die App sieht sie nie.

## Probleme & Lösungen

- **„Could not find the function public.delete_own_account … in the schema cache"** beim Klick auf *Account löschen*: Die RPC-Funktion fehlt in deiner Datenbank, oder PostgREST hat sie noch nicht geladen. Führe im **SQL Editor** die Datei **`fix-delete-account.sql`** aus – sie legt die Funktion (erneut) an und löst mit `NOTIFY pgrst, 'reload schema'` ein Neuladen des Schema-Caches aus. Danach funktioniert *Account löschen* sofort (kein Neustart nötig). Wer das komplette `supabase-setup.sql` nach dem 17.11.2026 neu ausführt, hat diesen Schritt bereits enthalten.
- **„permission denied for table users"** beim Löschen (selten, nur bei sehr neuen Supabase-Projekten): einmalig `grant delete on auth.users to postgres;` im SQL Editor ausführen, dann `fix-delete-account.sql` erneut laufen lassen.
