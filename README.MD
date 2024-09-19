# Onlineshop für PostgreSQL

**Das Projekt startete ich am 13.09.2024**
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

## Physisches ERD und Implementierung der ersten Tabelle
Da ich ein logisches ERD vorgegeben bekommen habe, möchte ich es zunächst richtig verstehen, Unklarheiten klären und überlegen, welche Datentypen hier richtig wären, bevor ich direkt implementiere.

**HINWEIS**
Dies ist das erste Mal, dass ich mit Postgres eine Datenbank erstelle, und auch, dass ich mich mit Onlineshops auseinandersetze. Deshalb mache ich Fehler, aus denen ich lerne. Besonders die Datentypen, die es bei MySQL/MariaDB und SQLite nicht gibt, sind also neu und ich möchte sie ausprobieren. Im ERD habe ich die erste Tabelle (Benutzerkonto) angelegt und wollte sie sofort erstellen, um auszuprobieren, ob alles so klappt. 
Das hat es auch technisch, aber später fand ich noch kleine Dinge heraus bzw. lernte noch etwas über die Datentypen, das ich hier dokumentiere.
Nach der ersten Tabelle habe ich das restliche ERD komplett erstellt. Mir ist bewusst, dass ich noch mehr Sachen lernen und feststellen werde. Um es anschaulich zu machen, werde ich das ERD vor und nach dem Implementieren mit Korrekturen zeigen. Die Überlegungen dazu schreibe ich in die Dokumentation.


### Benutzerkonto

Benutzer können einen Benutzernamen anlegen. Dieser sollte nicht zweimal vorkommen, deswegen ist er unique. Dasselbe gilt für die E-Mail-Adresse, die es gar nicht doppelt geben kann.
Für die Adresse wähle ich einen Composite Data Type, der den vollständigen Namen, die Straße + Hausnummer und die Postleitzahl enthalten soll. Bis auf Weiteres kann man nur ins Inland bestellen. Die Rechnungsadresse wird von der Anwendung abgefragt, sollte sie von der Lieferadresse abweichen. Sie ist somit NULLABLE, aber die Lieferadresse ist auszufüllen, also NOT NULL. 

Für die Zahlungsart gebe ich drei Möglichkeiten vor, somit nutze ich hier den Datentyp enum. In der Anwendung könnte man dann durch ein Dropdown-Menü oder Multiple-Choice-Kästchen auswählen. Gibt man Abbuchung oder Kreditkarte an, erscheint ein Feld, in das man die Daten eintragen kann.



### Implementierung Benutzerkonto
```sql
-- zuerst erstelle ich die komplexen Datentypen
-- ENUM:
CREATE TYPE Zahlungsart AS ENUM ('abbuchung', 'kreditkarte', 'postleitzahl');

-- !EINSCHUB 16.09.2024

-- (danach fiel mir auf, was ich hier statt 'paypal' 'postleitzahl' eingegeben hatte.

ALTER TYPE Zahlungsart ADD VALUE 'paypal';

-- ich sucht nach Möglichkeiten, den Wert zu verändern und fand dazu zunächst nur ADD VALUE. Dann fand ich aber kein DELETE VALUE oder etwas dergleichen. Ich lernte, dass man keine Werte aus enums löschen kann, weil das die ganze Architektur der Datenbank zerstören kann, wenn man erst mal Daten in der Datenbank hat. Deswegen gibt es den Befehl gar nicht erst. Zum Glück. Sollte man als Shop später noch die Zahlungsart ändern, z. B. wenn Paypal sich nicht mehr rentieren sollte, entfernt man es aus dem Frontend, die Kunden können es nicht mehr auswählen, in den Datensätzen bleibt es aber erhalten. So bleibt die Integrität der Datenbank erhalten und später kann man in den alten Datensätzen sehen, womit damals bezahlt wurde.
-- Hat man sich, wie ich, verschrieben, kann man auch jederzeit den Namen der Spalte ändern. Bei einer genaueren Suche habe ich auch gefunden, wie es geht. Wegen o. g. Erklärungen würde ich jedoch für den Fall, dass ich irgendwann eine andere Methode hinzufügen und statt Paypal benutzen will, in jedem Fall einen neuen Wert dafrür ins Enum eintragen.
-- Ich führe jetzt einfach eine vierte Zahlungsart ein:
ALTER TYPE Zahlungsart RENAME VALUE 'postleitzahl' TO 'rechnung';
-- EINSCHUB ENDE


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

-- !17.09.2024
-- Best Practice ist es, die Tabellennamen in der Mehrzahl zu implementieren. Ideal ist es, das schon im Vorhinein festzulegen. Man kann die Tabellen aber auch umbenennen. Weil ich die erste Tabelle angelegt habe, bevor ich den Rest des ERDs gebaut habe, lasse ich den Namen so stehen. Am Ende des Projektes werde ich ein aktualisiertes ERD zeigen.

ALTER TABLE benutzerkonto
RENAME TO benutzer;

```
### die Tabellen Lager und Produkte

Zur Tabelle Lager ist anzumerken, dass ich mir gerade Gedanken gemacht habe, wie das mit der Lagerung überhaupt funktioniert. Grobe Recherchen ergaben, dass es die Möglichkeit einer zentralen oder dezentralen Lagerung gibt, mit dem Unterschied, dass man bei der zentralen Lagerung ein einziges Lager hat. (Spoiler: dann bräuchte ich jetzt aber irgendwie keine Tabelle für die Lager). Tatsächlich hbe ich auch gefunden, dass für Onlineshops die deznetrale Lagerung benutzt wird. Und weil ein Comicshop vielleicht zumindest am Anfang nicht so ein Riesengeschäft ist, dass man eigene Lagerhäuser unterhalten könnte, habe ich noch recherchiert, ob man einzelne Lagerräume anmieten kann. Kann man. Da bin ich aber beruhigt!

Fragen tun sich bei mir in der Beziehung zwischen Lager und Produkten auf. Im vorgegebenen Diagramm gibt es eine One-to-Many-Beziehung zwischen der Tabelle "Lager" und der Tabelle "Produkte". Wäre dies nun ein Auftrag für einen echten Shop, würde ich die Betreiber*innen fragen. Ich gehe nun davon aus, dass wir mehrere Lager haben und dass, aufgrund der kurzen Lieferwege, die der Vorteil eines dezentralen Lagerungssystems sind, möglichst in mehreren Lagern dasselbe Produkt vorrätig sein soll. Ein Produkt kann in mehreren Lagern sein und in einem Lager sind mehrere verschiedene Produkte. Das ist dann eine *Many-to-Many-Beziehung*. 
Hier sollte also noch eine *Hilfstabelle* implementiert werden. Das ist die Produkt-im-Lager-Tabelle. Zunächst dachte ich an einen Composite Primary Key auf der Produkt-Id und der Lager-Id und wunderte mich allgemein, wieso es keine Composite Keys in dem vorgegebenen Diagramm gibt. Weitere Überlegungen führten zu dem Ergebnis, dass ein Primary Key niemals NULL sein darf. Was ist aber, wenn das Lager umzieht oder ein Produkt vom Markt genommen wird? Wahrscheinlich kann aus verschiedenen Gründen etwas NULL werden. Eine zusätzliche Id als Primary Key ändert auch nichts in der Funktion der Tabelle, also lege ich keinen Composite Kay an. 
Die Tabelle bekommt eine weitere Spalte, die die Stückzahl eines Produktes in einem der Lager anzeigt. Zudem füge ich noch eine Spalte letzte_lieferung_am ein, die einen Timestamp erhält, der bei jeder Belieferung aktualisiert wird. So kann man sehen, welche Produkte zu Ladenhütern werden und vielleicht veräußert werden müssen, wenn es im Lager eng wird.

### ERD vorher
Hier ist mein physisches ERD, Stand 17.09.2024:

![physisches Entity-Relationship-Diagram](/ERD-physisch.png)



### Eilmeldung! ENUM sinnlos!
17.09.2024: Es ist alles sinnlos!!! Was ist passiert?
Nein, nicht alles ist sinnlos, ich muss nur eine Spalte in meiner Tabelle von einem ENUM zu einem normalen VARCHAR umstellen. Und das nicht, weil es sich bei ENUM um einen sinnlosen Datentyp handelt, sondern weil Zahlungsmethoden sich ändern könnten und man dann nicht jedes Mal die Programmiererin des Vertrauens rufen will, um die Zahlungsmethode in den Datentyp reinzuprokeln, damit man ihn in die Tabelle einpflegen kann. 
ENUM könnte man bei Spalten verwenden, die unverändert bleiben, z. B. wenn man in einem Lieferrestaurant anzeigen will, ob das Essen erst bestellt oder in Vorbereitung ist, oder schon ausgeliefert und bezahlt (wenn man das nicht mit einer elektronischen Zahlungsart schon vorher ...eeeeeehm!)

```sql
ALTER TABLE benutzer
ALTER COLUMN zahlungsart 
SET DATA TYPE VARCHAR(20);
```
Trotz eingefügten Datensatzes scheint es funktioniert zu haben. WIESO? Ich meine, cool! Allerdings muss ich jetzt doch was ausprobieren:

```sql
ALTER TABLE benutzer
ALTER COLUMN zahlungsart 
SET DATA TYPE INT;
```
Aha!

*ERROR:  column "zahlungsart" cannot be cast automatically to type integer
HINT:  You might need to specify "USING zahlungsart::integer".*
```sql
ALTER TABLE benutzer
ALTER COLUMN zahlungsart 
SET DATA TYPE INT USING zahlungsart::integer;
```
*ERROR:  invalid input syntax for type integer: "abbuchung"*

Mein eingepflegter Datensatz hat den String "abbuchung" in dem Feld der Spalte. Ich hatte Strings in den ENUM eingegeben und VARCHAR ist ein String mit einer festgelegten Anzal an Zeichen. Beides trifft auf den Eintrag in der Spalte zu und deswegen konnte ich sie nachträglich ändern.

Nichtsdestotrotz liegt da jetzt immer noch ein Datentyp ungenutzt im Speicher rum. Ich werde ihn mal löschen.
```sql
DROP TYPE zahlungsart;
```

## Tabellen implementieren

### Bestellungen
18.09.2024
Es gibt einen Grund, weshalb ich die Tabellen in einer bestimmten Reihenfolge baue. Die meisten enthalten Foreign Keys, die auf andere Tabellen verweisen. Diese Foreign Keys können aber nicht angelegt werden, wenn die Tabelle, auf die sie verweisen, nicht da ist. Man könnte sie zwar später hinzufügen, aber man kann es auch in der richtigen Reihenfolge machen, um nicht durcheinanderzukommen.

Die Tabelle benutzer ist da und bestellungen hat einen Foreign Key, der auf benutzer verweist. Ich lege sie als nächste Tabelle an.

```sql
CREATE TABLE bestellungen (
    id_bestellung SERIAL PRIMARY KEY,
    bestelldatum TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    fid_benutzer INT REFERENCES benutzer(id_benutzer),
    gesamtpreisnetto NUMERIC(10,2),
    mwst NUMERIC(10,2),
    gesamtpreisbrutto NUMERIC(10,2)
);

INSERT INTO bestellungen (fid_benutzer, gesamtpreisnetto, mwst, gesamtpreisbrutto) VALUES (1, 9.34, 0.65, 9.99);

-- sind die Tabellen verbunden?
SELECT ben.benutzername, best.bestelldatum 
FROM benutzer ben 
INNER JOIN bestellungen best ON ben.id_benutzer = best.fid_benutzer;

-- ergibt: 
 benutzername |        bestelldatum        
--------------+----------------------------
 user1        | 2024-09-18 14:04:39.375459
(1 row) 

-- die beiden Tabellen sind miteinander verbunden.
```

### Bestellhistorien
Diese Tabelle verweist auf die Tabellen Benutzer und Bestellungen. Allerdings kann sie auch für eine schnelle Übersicht sorgen, an welchem Tag wieviele Bestellungen getätigt wurden. Da das Bestelldatum bei Abfragen fast immer mit dieser Information zusammenhängen wird, wurde hier eine Denormalisierung vorgenommen, um auf einer Tabelle bleiben zu können, wenn man wissen möchte, wie viele Bestellungen an einem Tag getätigt wurden.

```sql
CREATE TABLE bestellhistorien (
    id_historie SERIAL PRIMARY KEY,
    fid_benutzer INT REFERENCES benutzer(id_benutzer),
    fid_bestellung INT REFERENCES bestellungen(id_bestellung),
    bestelldatum TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- wir tragen unsere Bestellung ein:
INSERT INTO bestellhistorien(fid_benutzer, fid_bestellung)
VALUES (1,1);

world_of_comics=# SELECT * FROM bestellhistorien;
 id_historie | fid_benutzer | fid_bestellung |        bestelldatum        
-------------+--------------+----------------+----------------------------
           1 |            1 |              1 | 2024-09-18 14:38:19.804368
(1 row)

-- natürlich sollte der Timestamp sowie die gesamte Bestellung zeitgleich in beide Tabellen eingetragen werden. User1 beschließt also, dass sie/er noch etwas vergessen hat ...

INSERT INTO bestellungen (fid_benutzer, gesamtpreisnetto, mwst, gesamtpreisbrutto) VALUES (1, 28.03, 1.96, 29.99); 

INSERT INTO bestellhistorien(fid_benutzer, fid_bestellung)
VALUES (1,2);

-- Ich habe beide Befehle gemeinsam abgesendet, doch eigentlich code ich die 2 nicht hart rein. In diesen Foreign Key soll die generierte id, die die Bestellung tatsächlich in ihrer Tabelle hat , eingetragen werden. 
-- Wenn ich später in der Anwendung die Bestellung abschicke, passieren dort mehrere Befehle hintereinander, in denen erst die Bestellung zusammen mit der id_benutzer in die Tabelle Bestellungen eingetragen wird. Sie bekommt eine Id_Bestellung, die durch die Anwendung ausgelesen und dann an die Tabelle Bestellhistorien übergeben wird. Der Timestamp wird in beiden Tabellen automatisch erzeugt.

world_of_comics=# SELECT * FROM bestellungen;
 id_bestellung |        bestelldatum        | fid_benutzer | gesamtpreisnetto | mwst | gesamtpreisbrutto 
---------------+----------------------------+--------------+------------------+------+-------------------
             1 | 2024-09-18 14:04:39.375459 |            1 |             9.34 | 0.65 |              9.99
             2 | 2024-09-18 14:46:06.968813 |            1 |            28.03 | 1.96 |             29.99
(2 rows)

world_of_comics=# SELECT * FROM bestellhistorien;
 id_historie | fid_benutzer | fid_bestellung |        bestelldatum        
-------------+--------------+----------------+----------------------------
           1 |            1 |              1 | 2024-09-18 14:38:19.804368
           2 |            1 |              2 | 2024-09-18 14:46:06.979294
(2 rows)

-- zu beachten ist, dass die Timestamps wenige Bruchteile einer Sekunde auseinanderliegen. Die Abfragen wurden nacheinander ausgeführt.

SELECT ben.benutzername, best.gesamtpreisbrutto 
FROM bestellhistorien his
INNER JOIN benutzer ben ON ben.id_benutzer = his.fid_benutzer
INNER JOIN bestellungen best ON best.id_bestellung = his.fid_bestellung;
 benutzername | gesamtpreisbrutto 
--------------+-------------------
 user1        |              9.99
 user1        |             29.99
(2 rows)

```
### Tabellen Produkte und Bestellposition
Bis jetzt hat User1 einfach nur bestellt und einen Preis eingegeben. Ich würde ja gern Witze darüber machen, dass User1 bis jetzt nur bezahlt, aber ncihts bekommen hat, aber für den Witz fehlt noch eine andere Tabelle.
Zahlen können wir später, jetzt wird erst mal der Bestellvorgang mit einer Tabelle für die Dinge, die bestellt werden können, komplett gemacht.
Dazu kommt auch die Tabelle Bestellpositionen, denn diese ist wieder eine Hilfstabelle, um eine Many-to-Many-Beziehung (Bestellungen können n Produkte enthalten und ein Produkt kann in n Bestellungen bestellt werden) aufzulösen.

Wieder fange ich mit der Tabelle Produkte an, da ich sie anschließend für den Foreign Key brauche.

```sql
CREATE TABLE produkte (
    id_produkt SERIAL PRIMARY KEY,
    name VARCHAR(100),
    isbn VARCHAR(13),
    preis NUMERIC(5, 2)
);

CREATE TABLE bestellpositionen (
    id_bestellposition SERIAL PRIMARY KEY,
    fid_bestellung INT REFERENCES bestellungen (id_bestellung),
    fid_produkt INT REFERENCES produkte (id_produkt),
    menge INT
);

-- Mir fällt auf, es geht um Bücher. Vielleicht sollte auch noch dabeistehen, wer es geschrieben hat und bei welchem Verlag es erschienen ist.

ALTER TABLE produkte ADD COLUMN autor VARCHAR(150);

ALTER TABLE produkte ADD COLUMN verlag VARCHAR(50);

-- außerdem haben die Produkte einen verschiedenen Mehrwrtsteuersatz, je nachdem, ob es sich um Bücher (7 %) oder Figuren und Spiele handelt (19 %)

ALTER TABLE produkte ADD COLUMN mwst_satz NUMERIC;
-- damit man damit auch rechnen kann, ist es jetzt ein NUMERIC
```

19.09.2024: Nach weiteren Überlegungen komme ich zu dem Ergebnis, dass ich in diese Tabelle den Burtto- und Nettopreis sowie den Betrag der Mehrwertsteuer haben möchte, damit die einzelnen Beträge auf Anwendungsebene  ausgerechnet werden können und in der Tabelle Bestellungen einfach nur aufaddiert werden können.

```sql
ALTER TABLE produkte RENAME COLUMN preis TO preis_netto;

ALTER TABLE produkte ADD COLUMN preis_brutto NUMERIC(10, 2);

ALTER TABLE produkte ADD COLUMN mwst_betrag NUMERIC(10, 2);


INSERT INTO produkte(name, isbn, autor, verlag, mwst_satz, preis_netto, mwst_betrag, preis_brutto) VALUES
('Frieren 10', '9783753919973', 'Yamada/Abe', 'altraverse', 7, 9.35, 0.65, 10),
('Der Verkehrte Himmel', '9783964451088', 'Ross', 'avant-verlag', 7, 26.17, 1.83, 28),
('Kinderland', '9783943143904', 'Mawil', 'Reprodukt', 7, 27.10, 1.90, 29),
('Sonne und Beton', '9783446269613', 'Lobrecht/Haus', 'hanserblau', 7, 16.82, 1.18, 18),
('Wie ein leeres Blatt', '9783551713889', 'Bloulet/Bagieu', 'Carlsen', 7, 11.21, 0.79, 12),
('Naruto Shippuden - Manga-Figur 17cm', '3296580369089', 'Anime Heroes', 'BANZAI', 19, 23.95, 4.55, 28.50);

-- wir haben zwei Bestellungen, denen ich jetzt Produkte zuordne

INSERT INTO bestellpositionen (fid_bestellung, fid_produkt, menge) VALUES
(1, 6, 1),
(2, 1, 2),
(2, 2, 1);

-- jetzt stimmen bei den Bestellungen die Geldbeträge nicht mehr, weil ich mir für die Testdaten welche ausgedacht habe. 

UPDATE bestellungen SET gesamtpreisnetto = 23.95, mwst = 4.55, gesamtpreisbrutto = 28.50 WHERE id_bestellung = 1;

-- Die Besstellung Nr. 2 enthält 2x den Manga à 10 Euro und einen Comic à 28 Euro, also Gesamtpreisbrutto 48.00

UPDATE bestellungen SET gesamtpreisnetto = 40.34, mwst = 7.66, gesamtpreisbrutto = 48.00 WHERE id_bestellung = 2;

SELECT * FROM bestellungen;

 id_bestellung |        bestelldatum        | fid_benutzer | gesamtpreisnetto | mwst | gesamtpreisbrutto 
---------------+----------------------------+--------------+------------------+------+-------------------
             1 | 2024-09-18 14:04:39.375459 |            1 |            23.95 | 4.55 |             28.50
             2 | 2024-09-18 14:46:06.968813 |            1 |            40.34 | 7.66 |             48.00
(2 rows)
```



### Retourenprodukte
18.09.2024 Festgestellt, dass hier auf jeden Fall die Bestellposition rein muss, nicht (unbedingt) die id des Produktes. Die kann man als Denormalisierung aber einsetzen, dann muss man das Produkt nicht über einen JOIN später suchen.

