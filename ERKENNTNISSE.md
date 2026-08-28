# Was wir beim Übersetzen von Pokémon Unbound gelernt haben

Notizen aus der deutschen Übersetzung von **Pokémon Unbound 2.1.1.1**, einem
GBA-ROM-Hack auf FireRed-Basis mit CFRU. Gedacht für alle, die dasselbe
vorhaben, egal für welche Sprache.

Der rote Faden: **Fast jeder ernste Fehler kam nicht vom Übersetzen, sondern
vom Auslesen.** Der Text war richtig übersetzt und byteweise korrekt
geschrieben. Nur war die Stelle, an die er geschrieben wurde, gar keine
Textstelle.

---

## 1. Der Dump ist der Flaschenhals, nicht die Übersetzung

Ein Auslesewerkzeug entscheidet bei jedem Byte, ob dort Text steht. Jede
Faustregel dabei kostet Treffer. Was uns die einzelnen Regeln gekostet haben:

| Annahme im Dump | übersehene Stellen |
| --- | --- |
| Zeiger stehen an durch 4 teilbaren Adressen | 5481 |
| Text hat mindestens 8 sichtbare Zeichen | 2094 |
| Was Steuerbefehle enthält, ist binär | 9596 |
| Der Buchstabenanteil misst führende Leerzeichen mit | 12 |
| Ein Text besteht überwiegend aus Buchstaben | 49 |
| Text steht immer hinter einem Zeiger | 221 |
| Jeder Text kommt nur einmal vor | 267 |

**Zeiger byteweise suchen.** CFRU bettet Skripte ein, und deren
`loadpointer`-Operanden stehen oft schief. Wer nur jede vierte Adresse liest,
verliert Tausende Stellen. Bei uns war die komplette Einführung darunter, was
erst auffiel, weil der erste Bildschirm englisch blieb.

**Steuerbefehle nicht als Binärmüll abtun.** Ein Drittel aller Dialoge enthält
Farbcodes.

**Feste Felder haben keinen Zeiger.** Die Art-Kategorien wie „Snow Hat
Pokémon" stehen inline in der Artenstruktur, mit Nullbytes aufgefüllt. Wer nur
Text hinter einem Abschlussbyte sucht, findet keine einzige davon.

**Rahmen bestehen fast nur aus Platzhaltern.** Die Statusseite baut ihren Satz
aus vierzehn Varianten. Zwischen den Einsetzmarkern stehen zwei englische
Wörter. Jeder Buchstabenfilter wirft das weg.

---

## 2. Nicht alles hinter einem Abschlussbyte ist Text

Das ist die gefährlichste Lücke, weil sie nicht Fehlendes bedeutet, sondern
**Zuviel**: Einträge, die es gar nicht geben dürfte. Solange die Übersetzung
ihren Inhalt zufällig unverändert lässt, fällt nichts auf. Ein einziges
geändertes Byte reicht für einen Absturz.

### Bewegungsdaten

Unser einziger echter Absturz. Bei `0x08750C1A` stehen 29 Bytes
Bewegungsbefehle, bevor der Satz beginnt:

```
0e 0c 0c 0c 0f 0f 0f 04 fe aa aa aa aa aa 02 fe ...
```

Davor steht ein Abschlussbyte, also hielt der Dump das für einen Textanfang.
Zwei der Bytes sind `0xFE`. **Im Text heißt `0xFE` Zeilenumbruch, in einer
Bewegungsliste Ende.** Die Übersetzung ersetzte die Umbrüche durch
Leerzeichen. Damit endete die Liste nie mehr, das Spiel arbeitete die
folgenden Bytes als Schrittbefehle ab und sprang irgendwann in Daten.

Der Absturz kam an einer ganz anderen Stelle als der Fehler, nämlich in einer
Zwischensequenz, in der sich Figuren bewegen. Gefunden haben wir ihn erst
durch neun Runden Halbieren, bis genau ein Eintrag übrig war.

### Skriptcode

Zwei Einträge begannen mitten im Skript, direkt auf einem `loadpointer`. Dort
zu schreiben zerstört eine Instruktion.

### Auffüllung vor einem Feld

Bei `0x08876676` stehen zwei Nullbytes und dann `Tafelwasser`. Der echte
Artikelname beginnt zwei Bytes später. Der Dump legte beide als Einträge an.
Geschrieben wurde die frühere Fassung, mit einem Leerzeichen zu wenig. Der
Name rutschte ein Byte nach links, und wer ihn über seine eigene Adresse las,
bekam **`afelwasser`**.

### Erkennungsmerkmale, die sich bewährt haben

1. **Drei oder mehr seltene Sonderzeichen** (Bytes `0x01` bis `0x1F`, im Spiel
   Akzentbuchstaben) unter den ersten zwölf echten Zeichen.
2. **Ein `loadpointer` in den ersten Bytes.**
3. **Der Originaltext passt nicht in sein eigenes Fach.** Dann hat die
   Platzberechnung ein `0xFF` erwischt, das kein Abschlussbyte ist, sondern das
   Parameterbyte eines Platzhalters.
4. **Der Eintrag beginnt mit Nullbytes**, und am ersten echten Zeichen beginnt
   ein anderer Eintrag oder eine Dublette mit weniger Auffüllung.

Beim Zählen für Regel 1 **müssen Steuerbefehle übersprungen werden**. `0xFC`,
`0xFD` und die Präfixe `0xF8`, `0xF9` bringen Parameterbytes mit, die im
Bereich der Sonderzeichen liegen. Ohne diese Ausnahme wurden aus 7 Treffern
295 Fehlalarme.

Eine Regel haben wir wieder verworfen: „vier Bytes, die wie eine ROM-Adresse
aussehen, also Datenstruktur". Sie hat 22 echte Texte angeschwärzt. In einem
Satz wie `Jax:` gefolgt von einem Farbcode ergeben die Bytes ab Position vier
zufällig eine gültige Adresse. Vier Byte, die wie eine Adresse aussehen,
kommen in 32 MB ständig vor.

---

## 3. Scherben: Einträge, die nie angezeigt werden, aber blockieren

Der Dump setzt an manchen Stellen mehrfach an, einmal am Textanfang und noch
einmal mittendrin. Die zweite Fassung beginnt dann im Wort.

Zeigt kein Zeiger darauf, kann das Spiel sie nie anzeigen. Schaden richtet sie
trotzdem an: Der Schreiber darf den äußeren Text nur bis zu ihrem Anfang
schreiben, sonst würde er sie überschreiben. Ein längerer Satz passt dann nicht
mehr und bleibt in der Ausgangssprache.

Bei uns: **1115 Stück.** Sie zu sperren hat 487 Umzüge erspart, und Verschieben
ist der riskanteste Schreibweg.

Zwei Merkmale müssen **zusammen** zutreffen, sonst wird es gefährlich:

1. Der Eintrag beginnt innerhalb des Fachs eines anderen.
2. Auf ihn zeigt kein brauchbarer Zeiger.

Eines allein reicht nicht. Viele echte Texte in Tabellen fester Breite haben
keinen Zeiger.

Und: Ein Eintrag, der selbst schon als „kein Text" gesperrt ist, darf nicht als
äußerer gelten. Sonst sperrt die Regel den echten Satz, der hinter dem Müll
steckt. Genau das ist uns bei `Tafelwasser` passiert.

---

## 4. Vier Wege, Text zu schreiben, in dieser Reihenfolge

**0. Gar nicht.** Zwei Sperrlisten, beide erzeugt statt gepflegt. Was kein Text
ist, wird nie beschrieben, unabhängig von jedem Schalter.

**1. An Ort und Stelle.** Sicher, weil der verfügbare Platz per Definition die
Länge des Originals bis zu seinem Abschlussbyte ist. Ein Text dort wird nur
kürzer, nie länger, und kann nichts überschreiben, was ihm nicht gehört.

**2. Kürzen, damit Weg 1 möglich wird.** Ohne diese Runden wäre nichts von dem
möglich gewesen, was danach kam.

**3. Verschieben.** Nur mit einer ausdrücklichen Adressliste, portionsweise
geprüft, nach jeder Stufe ein Testlauf im Spiel.

### Was beim Verschieben schiefgehen kann

**Ein vermeintlicher Zeiger ist Programmcode.** Dann startet das ROM nicht
mehr. Nur zwei Sorten anfassen: ausgerichtete Zeiger, und schiefe mit
`loadpointer` (`0x0F`) zwei Bytes davor. Alles andere ist eine Zahl, die
zufällig wie eine Adresse aussieht.

**Eine Stelle wird über zwei Wege erreicht**, und nur einer läuft über einen
Zeiger. Dann erscheint derselbe Text im selben Bau einmal übersetzt und einmal
nicht.

**Der Zielblock ist gar nicht frei.** Eine Folge von `0xFF` ist nicht
automatisch ungenutzt. Bei uns zeigte eine Datentabelle in den größten
`0xFF`-Block hinein. Jeden Block an ausgerichteten Verweisen auftrennen und
dahinter Sicherheitsabstand lassen.

---

## 5. Der Platz ist größer, als das Abschlussbyte sagt

Tabellen mit fester Feldbreite füllen den Rest hinter dem Abschlussbyte mit
`0x00` auf. In der Typentabelle sind es sieben Byte je Feld; `BUG` belegt vier
davon. Wer nur bis zum Abschlussbyte rechnet, hat drei Zeichen für das deutsche
Wort statt sechs, und `KÄFER` passt nie.

Die Grenze ist der **nächste bekannte Eintrag**, nicht das Ende der Nullen. Nur
bis dahin ist sicher, dass die Nullen zu diesem Feld gehören.

Vorher stand dort `FEUR` statt `FEUER`, `GRÜN` statt `GRAS` und ein
unübersetztes `BUG`.

---

## 6. Zwei Grenzen, nicht eine

Die **Bytezahl** entscheidet, ob der Text ins ROM passt. Die **Zeilenlänge**
entscheidet, ob er auf den Bildschirm passt. Ein Text kann die eine einhalten
und die andere reißen.

Verschieben schafft Platz im ROM, aber nicht im Fenster. Als wir `FREILASSEN`
in ein Menü schreiben wollten, passte es in den ROM-Platz und wäre trotzdem aus
der Box geragt.

Ein brauchbares Maß für die Fensterbreite: die breiteste Zeile der Texte, die
im ROM **beieinander** liegen. Solche Gruppen teilen sich in aller Regel ein
Fenster.

---

## 7. Zusammengesetzte Meldungen

Das Spiel baut Sätze aus Bausteinen. Jeder Baustein für sich richtig übersetzt,
der Satz trotzdem falsch.

**Kampfmeldungen.** Bei uns kam `Larvitars Vert. ab!` heraus. Das englische
`fell!` ist ein Vollverb, `ab!` nur eine Partikel. Gelöst, indem das
Ausrufezeichen in den Rahmen wanderte und die Aussagen `sank` und `stieg`
heißen.

**Platzhalter am Satzende.** Steht im Original ein Platzhalter am Ende ohne
Satzzeichen, bringt der Puffer das Satzzeichen mit. Der übersetzte Satz muss
dann ebenfalls dort enden, sonst steht `setzt Silberblick! ein`.

**Leerzeichen am Rand.** `The opposing ` braucht sein Leerzeichen, sonst steht
`Der gegnerischePionskora`. `Yes   ` benutzt seine Leerzeichen, um das
Auswahlfeld zu bemaßen; ohne sie schrumpft das Menü auf einen Buchstaben je
Eintrag.

**Ein verlorenes Leerzeichen am Anfang ist kein Schönheitsfehler.** Steht
dahinter ein weiterer Eintrag, verschiebt sich alles Nachfolgende um genau so
viele Bytes.

**Verlorene Zeilenumbrüche** sind harmlos, solange kein Platzhalter im Text
steht. Mit Platzhalter weiß niemand vorher, wie breit die Zeile wird.

---

## 8. Ein Wort, zwei Bedeutungen

Der Bestand kennt das Wort, nicht die Stelle, an der es steht.

`Withdraw` ist im Spiel zweierlei: eine Attacke, deutsch „Panzerschutz", und
der PC-Menüpunkt „Entnehmen". Übersetzt wurde die Attacke. Im PC-Menü stand
dann **`Panzer.`**

Umgekehrt genauso: `Valley Cave` steht in der Vokabelliste des
Einfach-Text-Systems, wo alles in Großbuchstaben geschrieben wird. Dieselbe
Zeichenfolge ist an anderer Stelle ein normaler Ortsname im Fließtext. Ein
Eintrag für beide Stellen macht eine davon falsch.

Wer das sauber lösen will, braucht **einen Eintrag je Fundstelle**, nicht je
Zeichenfolge.

---

## 9. Was gar kein Text ist

Nicht alles, was auf dem Bildschirm nach Schrift aussieht, ist Schrift.

**Typ-Abzeichen** wie `FIGHT` oder `DRAGON` sind Bildkacheln. Die Texttabelle
der Typen daneben ist übersetzbar und wird auch benutzt, nur eben nicht dort.

**`POWER` und `ACCURACY`** im Attackenmenü ebenfalls. `ACCURACY` kommt im
ganzen ROM kein einziges Mal als Text vor.

Ein einfacher Test: das Wort im ROM als Text suchen. Findet man es nicht, ist
es Grafik.

---

## 10. Was wie ein Fehler aussieht und keiner ist

**`Lake`** als Attackenname. Das ist der offizielle deutsche Name von *Brine*,
von Pökellake.

**Fehlende Kommas.** Bei kleiner Schrift ist ein Komma ein bis zwei Pixel
breit und geht auf hochskalierten Bildschirmfotos optisch unter. Das Byte
steht im ROM, und das Original benutzt an derselben Stelle dasselbe Byte. Vor
dem Suchen erst prüfen, ob die Ausgangsfassung es anders macht.

---

## 11. Der CRC32 einer UPS-Datei sagt nichts

Wer eine Patchdatei veröffentlicht, gibt gern ihre Prüfsumme mit dazu. Bei
UPS ist das wertlos: **Jede gültige UPS-Datei hat denselben CRC32,
`2144DF1C`.**

Der Grund steckt im Format. Eine UPS-Datei endet mit der Prüfsumme ihres
eigenen Inhalts. Hängt man an Daten deren CRC32 an, ergibt die Prüfsumme des
Ganzen immer denselben festen Wert. Das ist eine Eigenschaft des Verfahrens,
kein Zufall.

Nachgeprüft an zwei völlig verschiedenen Dateien: Unbounds offiziellem Patch
mit 26 MB und unserem mit 1,4 MB. Beide `2144DF1C`.

Aufgefallen ist es, weil zwei Patchdateien unterschiedlicher Größe dieselbe
Prüfsumme hatten. Das sah nach einem Fehler im eigenen Werkzeug aus.

Für die Veröffentlichung also **MD5 oder SHA256** nehmen. Und die Prüfsumme
des Ergebnisses angeben, denn die ist es, die den Nutzer interessiert.

---

## 12. Spielstände und Savestates

**Savestates verfälschen jeden Test.** Wer jedem neuen Bau denselben
Dateinamen gibt, damit der Spielstand passt, bekommt beim Start einen
Speicherabzug aus einem anderen ROM zurück, mit Adressen, die dort nicht mehr
stimmen. Das erzeugt genau dasselbe Fehlerbild wie ein echter Fehler im Bau.
Zwei unserer Testergebnisse waren dadurch wertlos.

Automatisches Laden **und** Speichern abschalten, sonst legt das Gerät beim
Beenden sofort einen neuen an.

**`.sav` und `.srm` sind dasselbe.** Roher Abzug des Batteriespeichers, 128
KiB, 32 Sektoren zu 4 KiB, Fußleiste bei `0xFF4`. Unbound benutzt eine eigene
Signatur, `0x01121999`, nicht die `0x08012025` der offiziellen Spiele. Der
einzige Unterschied zwischen den Formen sind 16 Bytes am Ende: die Echtzeituhr
als BCD.

**Vorsicht:** Eine frisch angelegte Datei ist 128 KiB lauter `0xFF` und enthält
**keinen** Spielstand. Sie sieht genauso groß aus wie eine echte.

**Was im Spielstand steht und was nicht.** Fundort und Level stehen dort als
Zahl; der angezeigte Satz wird bei jeder Anzeige neu gebaut. Ein bereits
gefangenes Pokémon zeigt nach dem Patchen also den übersetzten Satz.

Der **Spitzname** dagegen wird beim Fangen fest hineingeschrieben. In dieser
Spielgeneration gibt es kein Merkmal „hat einen Spitznamen"; angezeigt wird
immer genau das, was im Spielstand steht. Ein vor dem Patchen gefangenes
Pokémon behält deshalb seinen alten Artnamen für immer.

---

## 13. Vorgehen, das sich bewährt hat

**Der belastbare Maßstab ist das gebaute ROM, nicht der Bestand.**
Bestandsseitige Prüfungen vergleichen den Bestand mit sich selbst. Wir messen,
wieviel Text der Ausgangssprache im gebauten ROM über einen gültigen Zeiger
noch erreichbar ist. Diese Zahl fiel von rund 9000 über 750 und 276 auf 16.

**Jede Rechnung nur einmal.** Ein Prüfwerkzeug hatte die Zeigerregel des
Schreibers nachgebaut statt sie zu benutzen. Als sich die echte Regel änderte,
wichen beide voneinander ab, und eine Textstelle fiel durch beide Netze. Zwei
Fassungen derselben Rechnung sind immer eine zu viel.

**Jedes geänderte Byte muss sich erklären lassen.** Wir ordnen jede Änderung
im fertigen ROM einer erlaubten Kategorie zu: Textfach, vormals leerer Bereich,
nachgezogene Zeigerstelle. 1,25 Millionen geänderte Bytes, null unerklärte.
Das schließt „irgendwo danebengeschrieben" für das ganze ROM aus, nicht nur für
die gerade untersuchte Stelle.

Aber: Diese Prüfung erklärt eine Stelle für erlaubt, wenn sie in einem Textfach
liegt. Ist das Fach gar keins, winkt sie durch. **Sie ersetzt keine
Plausibilitätsprüfung des Bestands.**

**Halbieren schlägt Disassemblieren.** Als der Absturz sich statisch nicht
finden ließ, haben neun Baue mit je einer Hälfte des Bestands ihn auf einen
Eintrag eingegrenzt. Dazu ein Werkzeug, das eine Runde baut, und eines, das
Buch führt. Ein Vorzeichenfehler beim Halbieren fällt sonst erst zwei Runden
später auf, und dann sind zwei Testdurchgänge verloren.

**Beim Kürzen die Platzhalter zählen lassen.** Bei drei Kampfmeldungen ist uns
beim Kürzen ein Platzhalter verlorengegangen. Im Kampf hätte dort der falsche
Name gestanden. Eine Prüfung, die Platzhalter zwischen Original und Übersetzung
vergleicht, fängt das ab.

**Bei viel Steuerbytes den Rundlauf prüfen.** Lässt sich der Originaltext aus
seiner ausgelesenen Fassung wieder unverändert erzeugen? Wenn nicht, hat das
Auslesen etwas missverstanden, und die Übersetzung baut darauf auf.

---

## Zahlen zum Schluss

| | |
| --- | --- |
| Bestand | 19.830 Einträge |
| Textstellen im ROM | 21.793 |
| davon geschrieben | 20.966 |
| gesperrt, weil kein Text | 809 |
| noch englisch, weil zu lang | 18 |
| englischer Text noch erreichbar | 16 |

Die 18 sind kein Kürzungsproblem: Dort beginnt mitten im Satz ein zweiter
Eintrag, den das Spiel ebenfalls anspringt, und für die Übersetzung bleiben ein
bis zwei Bytes.

---

Eine gekürzte englische Fassung der wichtigsten Punkte liegt in
`reddit_antwort.md`.
