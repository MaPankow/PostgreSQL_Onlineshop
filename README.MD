# Onlineshop für PostgreSQL
In diesem Projekt möchte ich eine Postgres-Datenbank für einen Onlineshop anlegen. Ein logisches ERD ist bereits vorgegeben.
Für mein Beispiel wähle ich einen Online-Comicshop und nenne ihn World of Comics.
Für weitere Überlegungen und um Datentypen zu recherchieren und festzuhalten, lege ich noch ein physisches ERD an.


## Datenbank anlegen
Weil man das ja schon mal machen kann ...

```sql
-- Terminal-Befehle zum Anlegen einer neuen Datenbank
sudo -i -u postgres
psql -U postgres
CREATE DATABASE World_of_Comics;
-- Hinweis: Am besten Datenbank in lower case anlegen, es wird nämlich umgewandelt und der Befehl \c ist case sensitive, d. h. \c World_of_Comics wirft einen Fehler.
-- Die Datenbank heißt also world_of_comics
\c world_of_comics;
```

## Physisches ERD und Implementierung
Da ich ein logisches ERD vorgegeben bekommen habe, möchte ich es zunächst richtig verstehen, Unklarheiten klären und überlegen, welche Datentypen hier richtig wären, bevor ich direkt implementiere.

### Benutzerkonto

Benutzer können einen Benutzernamen anlegen. Dieser sollte nicht zweimal vorkommen, deswegen ist er unique. Dasselbe gilt für die E-Mail-Adresse, die es gar nicht doppelt geben kann.
Für die Adresse wähle ich einen Composite Data Type, der den vollständigen Namen, die Straße + Hausnummer und die Postleitzahl enthalten soll. Bis auf Weiteres kann man nur ins Inland bestellen. Die Rechnungsadresse wird von der Anwendung abgefragt, sollte sie von der Lieferadresse abweichen. Sie ist somit NULLABLE, aber die Lieferadresse ist auszufüllen, also NOT NULL. 
Für die Zahlungsart gebe ich drei Möglichkeiten vor, somit nutze ich hier den Datentyp enum. In der Anwendung könnte man dann durch ein Dropdown-Menü oder Multiple-Choice-Kästchen auswählen. Gibt man Abbuchung oder Kreditkarte an, erscheint ein Feld, in das man die Daten eintragen kann.



#### **Implementierung Benutzerkonto**
```sql
-- zuerst erstelle ich die komplexen Datentypen
-- ENUM:
CREATE TYPE Zahlungsart AS ENUM ('abbuchung', 'kreditkarte', 'postleitzahl');

-- danach fiel mir auf, was ich hier statt 'paypal' eingegeben hatte

ALTER TYPE Zahlungsart ADD VALUE 'paypal';
-- ich sucht nach Möglichkeiten, den Wert zu verändern und fand dazu zunächst nur ADD VALUE. Dann fand ich aber kein DELETE VALUE oder etwas dergleichen. Ich lernte, dass man keine Werte aus enums löschen kann, weil das die ganze Architektur der Datenbank zerstören kann, wenn man erst mal Daten in der Datenbank hat. Deswegen gibt es den Befehl gar nicht erst. Zum Glück. Sollte man als Shop später noch die Zahlungsart ändern, z. B. wenn Paypal sich nicht mehr rentieren sollte, entfernt man es aus dem Frontend, die Kunden können es nicht mehr auswählen, in den Datensätzen bleibt es aber erhalten. So bleibt die Integrität der Datenbank erhalten und später kann man in den alten Datensätzen sehen, womit damals bezahlt wurde.
-- Hat man sich, wie ich, verschrieben, kann man auch jederzeit den Namen der Spalte ändern. Bei einer genaueren Suche habe ich auch gefunden, wie es geht. Wegen o. g. Erklärungen würde ich jedoch für den Fall, dass ich irgendwann eine andere Methode hinzufügen und statt Paypal benutzen will, in jedem Fall einen neuen Wert dafrür ins Enum eintragen.
-- Ich führe jetzt einfach eine vierte Zahlungsart ein:
ALTER TYPE Zahlungsart RENAME VALUE 'postleitzahl' TO 'rechnung';

-- COMPOSITE TYPE 
CREATE TYPE Lieferadresse AS (
    name_komplett VARCHAR(100),
    strasse_nr VARCHAR(100),
    postleitzahl CHAR(5)
);

CREATE TYPE Rechnungsadresse AS (
    name_komplett VARCHAR(100),
    strasse_nr VARCHAR(100),
    postleitzahl CHAR(5)
);

CREATE TYPE Girokonto AS (
    kontoinhaber VARCHAR(50),
    iban VARCHAR(22),
    bank VARCHAR(50)
);

CREATE TYPE Kreditkarte AS (
    kartennummer VARCHAR(16),
    ablaufdatum DATE,
    cvv CHAR(3)
);

-- nach der ganzen Vorarbeit kann ich jetzt endlich meine erste Tabelle bauen
CREATE TABLE benutzerkonto (
    id_benutzer SERIAL,
    benutzername VARCHAR(50) UNIQUE,
    passwort VARCHAR(255),
    email VARCHAR(50) UNIQUE,
    lieferadresse Lieferadresse,
    rechnungsadresse Rechnungsadresse,
    zahlungsart Zahlungsart,
    abbuchung Girokonto,
    kreditkarte Kreditkarte,
    PRIMARY KEY (id_benutzer)
);

INSERT INTO benutzerkonto (benutzername, passwort, email, lieferadresse.name_komplett, lieferadresse.strasse_nr, lieferadresse.postleitzahl, zahlungsart, abbuchung.kontoinhaber, abbuchung.iban, abbuchung.bank) VALUES
('user1', 'sicheres-passwort', 'varchar1@email.de', 'Martina Musterfrau', 'Musterstraße 31', '15362', 'abbuchung', 'Martina Musterfrau', 'DE37800000000000000000', 'Spaßkiste');

-- ich habe NOT NULL vergessen
ALTER TABLE benutzerkonto ALTER COLUMN lieferadresse SET NOT NULL;
ALTER TABLE benutzerkonto ALTER COLUMN zahlungsart SET NOT NULL;
```

### Bestellhistorie
Eine Tabelle mit vielen Foreign-Keys, die auf die Bestellungen und Benutzer verweist und anzeigt, wann die Bestellung getätigt wurde.

#### **Implementierung Bestellhistorie**
