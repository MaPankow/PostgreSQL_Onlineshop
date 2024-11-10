```SQL

-- DB erstellen und erreichen

sudo -i -u postgres
psql -U postgres
CREATE DATABASE World_of_Comics;
\c world_of_comics;

-- Composite Datentypen erstellen

CREATE TYPE Lieferadresse AS (
    name_komplett VARCHAR(100),
    strasse_nr VARCHAR(100),
    postleitzahl INT
);

CREATE TYPE Rechnungsadresse AS (
    name_komplett VARCHAR(100),
    strasse_nr VARCHAR(100),
    postleitzahl INT
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

-- Tabellen erstellen 

CREATE TABLE benutzer (
    id_benutzer SERIAL,
    benutzername VARCHAR(50) UNIQUE,
    passwort VARCHAR(255),
    email VARCHAR(50) UNIQUE,
    lieferadresse Lieferadresse,
    rechnungsadresse Rechnungsadresse,
    zahlungsart VARCHAR(20),
    abbuchung Girokonto,
    kreditkarte Kreditkarte,
    PRIMARY KEY (id_benutzer)
);

ALTER TABLE benutzer ALTER COLUMN lieferadresse SET NOT NULL;
ALTER TABLE benutzer ALTER COLUMN zahlungsart SET NOT NULL;

CREATE TABLE bestellungen (
    id_bestellung SERIAL PRIMARY KEY,
    bestelldatum TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    fid_benutzer INT REFERENCES benutzer(id_benutzer),
    gesamtpreisnetto NUMERIC(10,2),
    mwst NUMERIC(10,2),
    gesamtpreisbrutto NUMERIC(10,2)
);

CREATE TABLE bestellhistorien (
    id_historie SERIAL PRIMARY KEY,
    fid_benutzer INT REFERENCES benutzer(id_benutzer),
    fid_bestellung INT REFERENCES bestellungen(id_bestellung),
    bestelldatum TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE produkte (
    id_produkt SERIAL PRIMARY KEY,
    name VARCHAR(100),
    autor VARCHAR(150),
    verlag VARCHAR(50),
    mwst_satz NUMERIC,
    isbn VARCHAR(13),
    preis_netto NUMERIC(5, 2),
    mwst_betrag NUMERIC(10, 2),
    preis_brutto NUMERIC(10, 2)
);

CREATE TABLE bestellpositionen (
    id_bestellposition SERIAL PRIMARY KEY,
    fid_bestellung INT REFERENCES bestellungen (id_bestellung),
    fid_produkt INT REFERENCES produkte (id_produkt),
    menge INT
);

CREATE TABLE zahlungen (
    id_zahlung SERIAL PRIMARY KEY,
    fid_bestellung INT REFERENCES bestellungen(id_bestellung),
    betrag NUMERIC (10,2),
    zahlungsdatum TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

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

CREATE TABLE lager (
    id_lager SERIAL PRIMARY KEY,
    standort VARCHAR(20),
    kapazitaet_m2 SMALLINT,
    einzugsgebiet_min INT,
    einzugsgebiet_max INT
);

CREATE TABLE produkt_im_lager (
    id_produkt_im_lager SERIAL PRIMARY KEY,
    fid_lager SMALLINT REFERENCES lager(id_lager),
    fid_produkt INT REFERENCES produkte(id_produkt),
    stueckzahl INT,
    letzte_lieferung TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- alle Tabellen auflisten lassen:
\dt

-- INSERTS (Beispiele)

INSERT INTO benutzer (benutzername, passwort, email, lieferadresse.name_komplett, lieferadresse.strasse_nr, lieferadresse.postleitzahl, zahlungsart, abbuchung.kontoinhaber, abbuchung.iban, abbuchung.bank) VALUES
('user1', 'sicheres-passwort', 'varchar1@email.de', 'Martina Musterfrau', 'Musterstraße 31', 15362, 'abbuchung', 'Martina Musterfrau', 'DE37800000000000000000', 'Spaßkiste');


INSERT INTO benutzer (benutzername, passwort, email, lieferadresse.name_komplett, lieferadresse.strasse_nr, lieferadresse.postleitzahl, zahlungsart, kreditkarte.kartennummer, kreditkarte.ablaufdatum, kreditkarte.cvv) VALUES
('user2', '1234', 'varchar2@email.de', 'Matthias Mustermann', 'Musterstraße 33', 25362, 'kreditkarte', '0123456789012345', '2025-01-01', '789');


INSERT INTO produkte(name, isbn, autor, verlag, mwst_satz, preis_netto, mwst_betrag, preis_brutto) VALUES
('Frieren 10', '9783753919973', 'Yamada/Abe', 'altraverse', 7, 9.35, 0.65, 10),
('Der Verkehrte Himmel', '9783964451088', 'Ross', 'avant-verlag', 7, 26.17, 1.83, 28),
('Kinderland', '9783943143904', 'Mawil', 'Reprodukt', 7, 27.10, 1.90, 29),
('Sonne und Beton', '9783446269613', 'Lobrecht/Haus', 'hanserblau', 7, 16.82, 1.18, 18),
('Wie ein leeres Blatt', '9783551713889', 'Bloulet/Bagieu', 'Carlsen', 7, 11.21, 0.79, 12),
('Naruto Shippuden - Manga-Figur 17cm', '3296580369089', 'Anime Heroes', 'BANZAI', 19, 23.95, 4.55, 28.50);

INSERT INTO lager (standort, kapazitaet_m2, einzugsgebiet_min, einzugsgebiet_max) VALUES
('Hamburg', 20, 20000, 29999),
('Berlin', 37, 00000, 19999),
('Köln', 25, 30000, 59999),
('Frankfurt', 20, 60000, 79999),
('München', 18, 80000, 99999);



-- Die Produkte werden aus der DB heraus geholt und im Frontend angezeigt wo man sie auswählen und die Preise zusammenrechnen lassen kann. Diese Daten werden mit eingetragen
-- user 2 bestellt id_produkt 2 einmal
INSERT INTO bestellungen (fid_benutzer, gesamtpreisnetto, mwst, gesamtpreisbrutto) VALUES
(2, 26.17, 1.83, 28);
-- die erste Bestellung bekommt die id 1; an dieser Stelle würde die id entsprechend mittels Backend ausgelesen und in bestellhistorien als fid eingetragen werden
INSERT INTO bestellhistorien (fid_benutzer, fid_bestellung) VALUES (2, 1);
INSERT INTO bestellpositionen (fid_bestellung, fid_produkt, menge) VALUES (1, 2, 1); 

-- user 2 zahlt
INSERT INTO zahlungen (fid_bestellung, betrag) VALUES
(1, 28);

SELECT l.standort, pl.stueckzahl FROM produkt_im_lager pl
INNER JOIN lager l ON pl.fid_lager = l.id_lager
WHERE pl.fid_produkt = 2 AND l.einzugsgebiet_min >= 20000 AND l.einzugsgebiet_max < 30000;

 standort | stueckzahl 
----------+------------
 Hamburg  |          5
(1 row)
-- 5 Stück sind noch auf Lager und eins davon wird an user 2 gesendet.
```