# Onlineshop für PostgreSQL
In diesem Projekt möchte ich eine Postgres-Datenbank für einen Onlineshop anlegen. Ein logisches ERD ist bereits vorgegeben.
Für mein Beispiel wähle ich einen Online-Comicshop und nenne ihn World of Comics.
Für weitere Überlegungen und um Datentypen zu recherchieren und festzuhalten, lege ich noch ein physisches ERD an.


## Datenbank anlegen
Weil man das ja schon mal machen kann ...

```sql
sudo -i -u postgres
psql -U postgres
CREATE DATABASE World_of_Comics;
-- Hinweis: Am besten Datenbank in lower case anlegen, es wird nämlich umgewandelt und der Befehl \c ist case sensitive, d. h. \c World_of_Comics wirft einen Fehler.
-- Die Datenbank heißt also world_of_comics
\c world_of_comics;
```

## Physisches ERD und Überlegungen
Da ich ein logisches ERD vorgegeben bekommen habe, möchte ich es zunächst richtig verstehen und überlegen, welche Datentypen hier richtig wären, bevor ich direkt implementiere.