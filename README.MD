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

### Tabelle Zahlungen
Jetzt sind alle Tabellen beisammen und so miteinander verknüpft, da können wir eine Rechnung schreiben.

```sql
CREATE TABLE zahlungen (
    id_zahlung SERIAL PRIMARY KEY,
    fid_bestellung INT REFERENCES bestellungen(id_bestellung),
    betrag NUMERIC (10,2),
    zahlungsdatum TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO zahlungen (fid_bestellung, betrag) VALUES
(1, 28.50),
(2, 48.00);
```


### Retouren und Retourenprodukte

20.09.2024 Festgestellt, dass bei den Retourenprodukten auf jeden Fall die Bestellposition rein muss, nicht (unbedingt) die id des Produktes. Denn die Retour ist Teil einer Bestellung gewesen. Die Produkt-Id kann man als Denormalisierung aber einsetzen, dann muss man das Produkt nicht über einen extra JOIN suchen und kommt schneller an den Erstattungsbetrag.

```sql
CREATE TABLE retouren (
    id_retour SERIAL PRIMARY KEY,
    fid_bestellung INT REFERENCES bestellungen(id_bestellung),
    fid_benutzer INT REFERENCES benutzer(id_benutzer),
    rueckgabedatum TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    erstattungsbetrag_netto NUMERIC(5, 2),
    mwst NUMERIC(5, 2),
    erstattungsbetrag_brutto NUMERIC(5, 2)
);

CREATE TABLE retourenprodukte (
    id_retourenprodukt SERIAL PRIMARY KEY,
    fid_retour INT REFERENCES retouren(id_retour),
    fid_produkt INT REFERENCES produkte(id_produkt),
    fid_bestellposition INT REFERENCES bestellpositionen (id_bestellposition),
    menge INT
);

-- Jetzt kann ich die Tabellen ausprobieren. User 1 hat zwei Mal denselben Manga bestellt. Sie/er schickt einen davon zurück.

INSERT INTO retouren (fid_bestellung, fid_benutzer, erstattungsbetrag_netto, mwst, erstattungsbetrag_brutto) VALUES
(2, 1, 9.35, 0.65, 10);

INSERT INTO retourenprodukte (fid_retour, fid_produkt, fid_bestellposition, menge) VALUES
(1, 1, 2, 1);

-- beide Inserts schicke ich gleichzeitig ab

world_of_comics=# SELECT * FROM retouren;
 id_retour | fid_bestellung | fid_benutzer |       rueckgabedatum       | erstattungsbetrag_netto | mwst | erstattungsbetrag_brutto 
-----------+----------------+--------------+----------------------------+-------------------------+------+--------------------------
         1 |              2 |            1 | 2024-09-20 11:35:22.041437 |                    9.35 | 0.65 |                    10.00
(1 row)


world_of_comics=# SELECT * FROM retourenprodukte;
 id_retourenprodukt | fid_retour | fid_produkt | fid_bestellposition | menge 
--------------------+------------+-------------+---------------------+-------
                  1 |          1 |           1 |                   2 |     1
(1 row)
```

###  Lager und Produkt_im_Lager
Ich nenne die Spalte für die Kapazität um in kapazitaet_m2, waas ausdrückt, dass die Quatdratmeterzahl eingetragen wird, und gebe ihr den Datentyp SMALLINT, weil ich nicht glaube, dass ein Lagerraum über 32,7 Quadratkilometer groß sein kann.
```sql
CREATE TABLE lager (
    id_lager SERIAL PRIMARY KEY,
    standort VARCHAR(20),
    kapazitaet_m2 SMALLINT
);

CREATE TABLE produkt_im_lager (
    id_produkt_im_lager SERIAL PRIMARY KEY,
    fid_lager SMALLINT REFERENCES lager(id_lager),
    fid_produkt INT REFERENCES produkte(id_produkt),
    stueckzahl INT,
    letzte_lieferung TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- auch hier nehme ich SMALLINT für den Foreign Key auf die Lager-Tabelle, da davon auszugehen ist, dass ein Comicshop nicht über 32767 Lager haben wird. (Falls er so weit expandieren sollte, können er es sich wohl leisten, den  Datentyp umstellen zu lassen)


-- vorerst könnte man Lagerboxen in Hamburg, Berlin, Köln, Frankfurt und München haben

INSERT INTO lager (standort, kapazitaet_m2) VALUES
('Hamburg', 20),
('Berlin', 37),
('Köln', 25),
('Frankfurt', 20),
('München', 18);

INSERT INTO produkt_im_lager (fid_lager, fid_produkt, stueckzahl) VALUES
(1, 1, 20), (1, 2, 5), (1, 3, 2), (1, 4, 0), (1, 5, 1), (1, 6, 10),
(2, 1, 15), (2, 2, 10), (2, 3, 15), (2, 4, 20), (2, 5, 20), (2, 6, 5),
(3, 1, 24), (3, 2, 10), (3, 3, 0), (3, 4, 10), (3, 5, 20), (3, 6, 10),
(4, 1, 0), (4, 2, 5), (4, 3, 10), (4, 4, 15), (4, 5, 1), (4, 6, 1),
(5, 1, 4), (5, 2, 13), (5, 3, 20), (5, 4, 25), (5, 5, 1), (5, 6, 10);

world_of_comics=# SELECT * FROM lager;
 id_lager | standort  | kapazitaet_m2 
----------+-----------+---------------
        1 | Hamburg   |            20
        2 | Berlin    |            37
        3 | Köln      |            25
        4 | Frankfurt |            20
        5 | München   |            18
(5 rows)

world_of_comics=# SELECT * FROM produkt_im_lager LIMIT 5;
 id_produkt_im_lager | fid_lager | fid_produkt | stueckzahl |     letzte_lieferung      
---------------------+-----------+-------------+------------+---------------------------
                   1 |         1 |           1 |         20 | 2024-09-20 13:42:01.86508
                   2 |         1 |           2 |          5 | 2024-09-20 13:42:01.86508
                   3 |         1 |           3 |          2 | 2024-09-20 13:42:01.86508
                   4 |         1 |           4 |          0 | 2024-09-20 13:42:01.86508
                   5 |         1 |           5 |          1 | 2024-09-20 13:42:01.86508
(5 rows)

```


## 5 Testabfragen
Die Datenbank ist fertig, aber ich möchte testen, ob alles korrekt miteinander verbunden ist. Deswegen stelle ich mir ein paar Testaufgaben.

### 1. Wie viele Produkte hat User1 bestellt und wie viel muss User 1 bezahlen? (Benutzername, Anzahl aller Produkte, die bestellt wurden, Gesamtpreisbrutto)

```sql
-- die Abfrage ist komplex, ich probiere erst mal Teile aus
SELECT SUM(gesamtpreisbrutto) FROM bestellungen WHERE fid_benutzer = 1;

  sum  
-------
 76.50
(1 row)

SELECT SUM(gesamtpreisbrutto) AS gesamtrechnung FROM bestellungen WHERE fid_benutzer = 1;

 gesamtrechnung 
----------------
          76.50
(1 row)

SELECT ben.benutzername, SUM(best.gesamtpreisbrutto) AS gesamtrechnung, SUM(pos.menge) AS anzahl_produkte FROM bestellungen best
INNER JOIN benutzer ben ON ben.id_benutzer = best.fid_benutzer
INNER JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung 
WHERE best.fid_benutzer = 1 
GROUP BY ben.benutzername;

 benutzername | gesamtrechnung | anzahl_produkte 
--------------+----------------+-----------------
 user1        |         124.50 |               4
(1 row)
--ups, was hat er denn hier ausgerechnet? 124.50 - 76,50 ist 48.00, die Summe der zweiten Bestellung. Das liegt daran, dass es 3 id_bestellposition gibt:

world_of_comics=# SELECT * FROM bestellpositionen;
 id_bestellposition | fid_bestellung | fid_produkt | menge 
--------------------+----------------+-------------+-------
                  1 |              1 |           6 |     1
                  2 |              2 |           1 |     2
                  3 |              2 |           2 |     1
(3 rows)

-- vielleicht lässt sich der Fehler in der Abfrage beheben, wenn ich nach der id_bestellung gruppiere?
SELECT ben.benutzername, SUM(best.gesamtpreisbrutto) AS gesamtrechnung, SUM(pos.menge) AS anzahl_produkte FROM bestellungen best
INNER JOIN benutzer ben ON ben.id_benutzer = best.fid_benutzer
INNER JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung 
WHERE best.fid_benutzer = 1 
GROUP BY best.id_bestellung;

ERROR:  column "ben.benutzername" must appear in the GROUP BY clause or be used in an aggregate function
LINE 1: SELECT ben.benutzername, SUM(best.gesamtpreisbrutto) AS gesa...

-- Menno! User1 hat als einziger Kunde eben zwei Mal bestellt!!! Ich lasse den benutzernamen erst mal raus.

SELECT SUM(best.gesamtpreisbrutto) AS gesamtrechnung, SUM(pos.menge) AS anzahl_produkte FROM bestellungen best
INNER JOIN benutzer ben ON ben.id_benutzer = best.fid_benutzer
INNER JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung 
WHERE best.fid_benutzer = 1 
GROUP BY best.id_bestellung;

 gesamtrechnung | anzahl_produkte 
----------------+-----------------
          28.50 |               1
          96.00 |               3
(2 rows)

-- trotzdem rechnet er immer noch die zweite Bestellung zwei mal, weil sie bei den Bestellpositionen 2x vorhanden ist. Vielleicht ist INNER JOIN die falsche Methode. Ich probiere einen LEFT JOIN aus, mit dem Gedanken, dass die Anzahl der Reihen in der rechten Tabelle dann egal wird:

SELECT SUM(best.gesamtpreisbrutto) AS gesamtrechnung, SUM(pos.menge) AS anzahl_produkte FROM bestellungen best
LEFT JOIN benutzer ben ON ben.id_benutzer = best.fid_benutzer
LEFT JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung 
WHERE best.fid_benutzer = 1 
GROUP BY best.id_bestellung;

 gesamtrechnung | anzahl_produkte 
----------------+-----------------
          28.50 |               1
          96.00 |               3
(2 rows)

-- hmmm, habe ich irgendwas in die Bestellungen insertet und vergessen?

world_of_comics=# SELECT * FROM bestellungen;
 id_bestellung |        bestelldatum        | fid_benutzer | gesamtpreisnetto | mwst | gesamtpreisbrutto 
---------------+----------------------------+--------------+------------------+------+-------------------
             1 | 2024-09-18 14:04:39.375459 |            1 |            23.95 | 4.55 |             28.50
             2 | 2024-09-18 14:46:06.968813 |            1 |            40.34 | 7.66 |             48.00
(2 rows)

-- es gibt zwei Einträge in den Bestellungen, aber drei Einträge in den Bestellpositionen, weil Bestellung 2 zwei verschiedene Produkte hat. Die Art des JOINS ändert daran nichts, dass die "beiden Bestellungen 2" zusammengerechnet werden. Mit DISTINCT kann man die Dopplung bei den bestellpositionen rausnehmen:

SELECT DISTINCT fid_bestellung FROM bestellpositionen;

 fid_bestellung 
----------------
              2
              1
(2 rows)

-- Idee:

SELECT SUM(menge) FROM bestellpositionen GROUP BY fid_bestellung;

 sum 
-----
   3
   1
(2 rows)

-- Aha! Ich gruppiere jetzt die gesamte Query über die fid_bestellung bei den Bestellpositionen

SELECT SUM(best.gesamtpreisbrutto) AS gesamtrechnung, SUM(pos.menge) AS anzahl_produkte FROM bestellungen best
INNER JOIN benutzer ben ON ben.id_benutzer = best.fid_benutzer
INNER JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung 
WHERE best.fid_benutzer = 1 
GROUP BY pos.fid_bestellung;

 gesamtrechnung | anzahl_produkte 
----------------+-----------------
          28.50 |               1
          96.00 |               3
(2 rows)

-- es nervt! Ich probiere weiter mit DISTINCT
SELECT DISTINCT fid_bestellung, menge FROM bestellpositionen;
 fid_bestellung | menge 
----------------+-------
              1 |     1
              2 |     2
              2 |     1
(3 rows)

SELECT DISTINCT fid_bestellung, SUM(menge) 
FROM bestellpositionen
GROUP BY fid_bestellung;

 fid_bestellung | sum 
----------------+-----
              1 |   1
              2 |   3
(2 rows)

SELECT SUM(best.gesamtpreisbrutto) AS gesamtrechnung, SUM(pos.menge) AS anzahl_produkte FROM bestellungen best
INNER JOIN benutzer ben ON ben.id_benutzer = best.fid_benutzer
INNER JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung 
WHERE best.fid_benutzer = 1 
GROUP BY (SELECT DISTINCT pos.fid_bestellung);

-- CHAT GPT schlägt wirren Code vor, weil es die Tabellen nicht kennt. Aber ausschlaggebend ist die Idee, einen INNER JOIN in die Subquery einzubauen. Ich nehme noch mal die Ausgangsquery: 

SELECT ben.benutzername, (SELECT SUM(best.gesamtpreisbrutto)) AS gesamtrechnung, (SELECT SUM(pos.menge)) AS anzahl_produkte 
FROM bestellungen best    
INNER JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung 
INNER JOIN benutzer ben ON ben.id_benutzer = best.fid_benutzer
GROUP BY ben.benutzername;

-- zerteilen:

SELECT SUM(pos.menge) AS anzahl_produkte FROM bestellungen best INNER JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung;

 anzahl_produkte 
-----------------
               4

SELECT ben.benutzername, (SELECT SUM(best.gesamtpreisbrutto)) AS gesamtrechnung, (SELECT SUM(pos.menge)  
    FROM bestellungen best 
    INNER JOIN bestellpositionen pos ON best.id_bestellung = pos.fid_bestellung) AS anzahl_produkte 
FROM bestellungen best
INNER JOIN benutzer ben ON ben.id_benutzer = best.fid_benutzer
GROUP BY ben.benutzername;

 benutzername | gesamtrechnung | anzahl_produkte 
--------------+----------------+-----------------
 user1        |          76.50 |               4
(1 row)

-- PAAAAAARTYYYYYYYYYYYY!!!

```





### 2. User1 bekommt eine Rechnung für Bestellung zwei. Dafür brauchen wir die Rechnungsadresse (in dem Fall die Lieferadresse), die Zahlungsart, den Nettogesamtbetrag, die Mwst und den Bruttogesamtbetrag.

```sql
SELECT ben.id_benutzer, ben.lieferadresse, ben.zahlungsart, best.gesamtpreisnetto, best.mwst, best.gesamtpreisbrutto
FROM benutzer ben
INNER JOIN bestellungen best ON best.fid_benutzer = ben.id_benutzer
WHERE id_bestellung = 2;

 id_benutzer |                 lieferadresse                  | zahlungsart | gesamtpreisnetto | mwst | gesamtpreisbrutto 
-------------+------------------------------------------------+-------------+------------------+------+-------------------
           1 | ("Martina Musterfrau","Musterstraße 31",15362) | abbuchung   |            40.34 | 7.66 |             48.00

```


### 3. Hat User 1 alle Rechnungen bezahlt? 

```sql
SELECT ben.id_benutzer, (SELECT COUNT(best.id_bestellung)) AS anzahl_bestellungen, (SELECT SUM(best.gesamtpreisbrutto)) AS gesamtrechnung, (SELECT SUM(z.betrag) 
    FROM bestellungen best
    INNER JOIN zahlungen z ON z.fid_bestellung = best.id_bestellung) AS summe_bezahlt
FROM bestellungen best 
INNER JOIN benutzer ben ON best.fid_benutzer = ben.id_benutzer
GROUP BY ben.id_benutzer;

 id_benutzer | anzahl_bestellungen | gesamtrechnung | summe_bezahlt 
-------------+---------------------+----------------+---------------
           1 |                   2 |          76.50 |         76.50
(1 row)
```







### 4. Welches Produkt (Name, Autor) hat User1 zurückgegeben und wie viel Geld muss zu welchem Konto zurückerstattet werden?

```sql
SELECT p.name, p.autor, p.preis_brutto AS rückerstattungsbetrag, ben.abbuchung AS zielkonto
FROM bestellpositionen pos 
INNER JOIN retourenprodukte rp ON rp.fid_bestellposition = pos.id_bestellposition
INNER JOIN produkte p ON rp.fid_produkt = p.id_produkt
INNER JOIN retouren r ON rp.fid_retour = r.id_retour
INNER JOIN benutzer ben ON ben.id_benutzer = r.fid_benutzer
WHERE id_benutzer = 1;

    name    |   autor    | rückerstattungsbetrag |                        zielkonto                        
------------+------------+-----------------------+---------------------------------------------------------
 Frieren 10 | Yamada/Abe |                 10.00 | ("Martina Musterfrau",DE37800000000000000000,Spaßkiste)
(1 row)

```


### 5. Welche Produkte (Name) sind in welchem Lager (Name) ausverkauft?

```sql
SELECT l.standort, p.name, p.isbn, pl.stueckzahl 
FROM produkt_im_lager pl 
INNER JOIN lager l ON l.id_lager = pl.fid_lager
INNER JOIN produkte p ON pl.fid_produkt = p.id_produkt
WHERE pl.stueckzahl = 0;

 standort  |      name       |     isbn      | stueckzahl 
-----------+-----------------+---------------+------------
 Hamburg   | Sonne und Beton | 9783446269613 |          0
 Köln      | Kinderland      | 9783943143904 |          0
 Frankfurt | Frieren 10      | 9783753919973 |          0
(3 rows)
```
## Physisches ERD nachher
Ein paar kleinere und größere Änderungen haben sich ergeben. Zum Überblick, wie die Datenbank dann aussieht, noch mal mein aktualisiertes physisches ERD:

![Entity-Relationship-Diagramm nachher](/ERD-physisch-nachher.png)
