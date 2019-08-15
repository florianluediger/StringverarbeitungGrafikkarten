# Effiziente String-Verarbeitung in Datenbankanfragen auf hochgradig paralleler Hardware

Dieses Repository enthält meine Master-Abschlussarbeit an der Technischen Universität Dortmund.
Der gesamte Code, der für das Umsetzen der Messungen angefertigt wurde, ist auf GitHub in [diesem Repository](https://github.com/florianluediger/GPULaneRefill) verfügbar.
Bei Fragen oder Anregungen bin ich per E-Mail erreichbar.
Um eine Idee von dem Thema zu bekommen folgt die Einleitung.

## Einleitung

Ein Großteil der Datensätze, die in realen Systemen zum Einsatz kommen, enthalten eine große Vielfalt an String-Datensätzen mit verschiedensten Eigenschaften.
Die Operationen, die auf diesen Datensätzen ausgeführt werden, reichen von Gleichheitsüberprüfungen über das Enthaltensein eines Teilstrings bis zum Erfüllen eines komplexen, regulären Ausdrucks.
In dieser Arbeit wird untersucht werden, wie sich die unterschiedlichen Operationen zur String-Verarbeitung verhalten, wenn diese auf einer hochgradig parallel arbeitenden Grafikkarte ausgeführt werden.
Außerdem wird analysiert, ob ein Verfahren zur Steigerung der Auslastung der Grafikkarte eine signifikante Leistungssteigerung erzielen kann.

### Motivation und Hintergrund

Die effiziente Berechnung unterschiedlicher Operatoren auf Zeichenketten ist essenziell für das Erreichen eines hohen Durchsatzes und für das Gewährleisten von maximaler Performanz.
In diesem Kontext versprechen Grafikkarten durch ihre hochgradig parallele Architektur in der Theorie eine bestmögliche Leistung.

Der Aufbau von moderner, hoch paralleler Hardware wie einer Grafikkarte führt dazu, dass diese besonders effizient mit gleichmäßigen Daten arbeiten kann.
Ein String-Datensatz ist dagegen typischerweise sehr heterogen durch die unterschiedlichen Längen der einzelnen Zeichenketten, wodurch das Verarbeiten auf einer Grafikkarte zunächst einige Schwierigkeiten birgt.
Aus diesem Grund verwenden bisherige Ansätze zur Verarbeitung von Zeichenketten das Konzept der Dictionaries [Mueller2014].
Dabei wird eine Tabelle mit allen Strings aufgebaut und zu diesen ein Schlüssel abgespeichert, welcher zusammen mit jedem String in den anderen Tabellen der Datenbank gespeichert wird.
Somit können String-Operationen durch andere Operationen auf den Schlüsseln abgebildet werden, wodurch diese eine einheitliche Struktur und damit ein effizientes Ausführungsmuster auf Grafikkarten erhalten.
Das Aufbauen und Verwalten des für diese Technik verwendeten Dictionaries erzeugt vor allem bei Daten, die sich häufig ändern, einen hohen Aufwand, wodurch die Leistungsfähigkeit des Gesamtsystems sinkt.
Um diesen Verwaltungsaufwand für eine zusätzliche Datenstruktur zu eliminieren, wäre es wünschenswert eine Lösung zu finden, die diesen Aufwand vermeidet und direkt auf den ursprünglichen String-Daten arbeitet.

Soll eine komplexere Operation auf den Daten ausgeführt werden, die mit regulären Ausdrücken arbeitet, ist die Verwendung eines Dictionaries nicht mehr möglich und spätestens an dieser Stelle muss auf eine andere Methode zurückgegriffen werden.
Die Verwendung von regulären Ausdrücken eröffnet in vielen Anwendungsfällen verschiedenste Möglichkeiten, String-Daten effizient zu verarbeiten und die Leistungsfähigkeiten des Datenbanksystems voll auszunutzen.
Reguläre Ausdrücke stellen ein mächtiges Werkzeug dar, welches den meisten Anwendern von Datenbankmanagementsystemen bekannt ist und vor allem relevant ist, da es das hoch effiziente Auswerten komplexer Muster erlaubt.

Die meisten aktuellen Datenbankmanagementsysteme bieten eine Unterstützung für reguläre Ausdrücke, da sie eine hohe Leistung beim Musterabgleich garantieren.
Gäbe es eine solche Funktion nicht, müsste eine entsprechende Selektion nach dem Ausführen des Anfrageplans durch das DBMS manuell im Anwendungsprogramm durchgeführt werden.
An dieser Stelle entstehen zahlreiche Probleme, da der Optimierer die Selektion nicht an der optimalen Stelle im Anfrageplan platzieren kann, damit frühzeitig Tupel weg fallen, die nicht in das Ergebnis aufgenommen werden.
Es entsteht also schon beim Datenbankserver eine erhöhte Last durch das unnötige Verarbeiten von Tupeln.
Diese werden zusätzlich noch über das Netzwerk zur Anwendung übertragen, wodurch erneut eine erhöhte Netzlast entsteht.
Die Anwendung wiederum muss anschließend mit einer potenziell geringeren Leistungsfähigkeit als der Datenbankserver die Selektion durchführen, wodurch an dieser Stelle wieder eine unnötig hohe Last entsteht.
Sollten also Anwendungsfälle auftreten, bei denen eine Selektion durch reguläre Ausdrücke gewünscht ist, stellt die Unterstützung dieser Operation durch das Datenbankmanagementsystem einen massiven Vorteil dar.
Auch für Datenbanksysteme, die auf Grafikkarten arbeiten sollte also sichergestellt sein, dass diese die maximal mögliche Performanz bei der Verarbeitung von regulären Ausdrücken erreichen.

### Zielsetzung

In dieser Arbeit sollen verschiedene Operationen, die auf String-Daten arbeiten, im Kontext von kompilierten Anfrageplänen mithilfe des Query Compilers DogQC untersucht werden.
Ziel ist es dabei die Performanz der umgesetzten Operatoren zu analysieren und Flaschenhälse, die durch eine schlechte Auslastung der Grafikkarte entstehen, zu identifizieren.
Diese Flaschenhälse sollen mithilfe des Lane Refill-Verfahrens eliminiert werden, wodurch eine Steigerung der Leistungsfähigkeit erreicht werden soll.
Neben einfachen String-Operatoren sollen auch komplexere Techniken, welche reguläre Ausdrücke zur String-Verarbeitung verwenden, analysiert werden und untersucht werden, ob diese einen Laufzeitgewinn durch das Lane Refill erreichen.
Mithilfe dieser Betrachtungen sollen Rückschlüsse darüber gezogen werden, ob das Lane Refill, welches bereits bei der Durchführung von Join-Operationen erfolgreich eingesetzt wird, auch für die Verarbeitung von Zeichenketten einen Nutzen bringt.

### Aufbau der Arbeit

Um verstehen zu können, warum bei der Verarbeitung von String-Daten auf Grafikkarten verschiedene Probleme auftreten können, werden zunächst der Grundaufbau und die wichtigsten Eigenschaften von Grafikprozessoren erklärt.
Anschließend werden die bei DogQC zum Einsatz kommenden Verfahren zur Kompilierung von Anfrageplänen erläutert und die Rahmenbedingungen für die nachfolgenden Untersuchungen festgelegt.
In Kapitel 4 wird der einfache String-Vergleich erarbeitet und auf verschiedene Probleme hingewiesen, die im nachfolgenden Kapitel durch den Einsatz des Lane Refill behoben werden.
An dieser Stelle wird außerdem erklärt, wie das Lane Refill-Verfahren funktioniert und warum es durch eine höhere Auslastung der Grafikkarte eine bessere Performanz erreichen kann.
Anschließend wird in Kapitel 6 erläutert, wie reguläre Ausdrücke mithilfe von endlichen Automaten ausgewertet werden können und ein geeigneter Ansatz für die Umsetzung im Query Compiler ausgewählt.
Im folgenden Teil wird der parallele Musterabgleich mit regulären Ausdrücken umgesetzt und ähnlich wie beim einfachen String-Vergleich durch das Lane Refill erweitert.
Vorbereitend auf die im letzten Teil der Arbeit durchgeführten Leistungstests der verschiedenen Algorithmen wird aufgezeigt, wie das Optimieren verschiedener Parameter bei der Ausführung von Algorithmen auf einer Grafikkarte einen Einfluss auf die Performanz des Systems nimmt und wie diese Optimierung durchgeführt werden kann.
Schließlich folgen zwei Kapitel zur Evaluation des einfachen String-Vergleichs und des Musterabgleichs mit regulären Ausdrücken, in denen untersucht wird, wie sich die unterschiedlichen Algorithmen verhalten und ob durch die Verwendung des Lane Refill-Verfahrens ein Laufzeitgewinn erreicht werden kann.
Abschließend wird in einem Fazit Stellung dazu genommen, ob die vorher beschriebene Zielsetzung erreicht werden konnte und ob das Lane Refill bei der Verarbeitung von Zeichenketten eine sinnvolle Anwendung findet.