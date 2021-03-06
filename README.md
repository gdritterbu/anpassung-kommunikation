# Anpassung des ausgewählten Kommunikationstools am Beispiel "Terminerstellung mit externen Teilnehmern"
Diese Dokumentation beschreibt die technische Implementierung einer Service-Orchestrierung des Prozesses “Terminerstellung mit externen Teilnehmern” mithilfe der Process Engine Camunda BPM. Im Kontext der Beschaffung einer Unternehmenskommunikationssoftware ist der Prozess “Terminerstellung mit externen Teilnehmern” Bestandteil eines fiktiven Unternehmens in Brandenburg an der Havel.

Ziel dieser Service-Orchestrierung ist die Umsetzung folgender Punkte:
- Umsetzung einer formularbasierte User-Aufgabe
- Umsetzung einer DMN-Regelaufgabe
- Umsetzung eines externen Service-Aufrufes
- Umsetzung einer Sendeaufgabe

## Verwendete Technologien
Für die technische Implementierung der Service-Orchestrierung wurden folgende Technologien verwendet:

- [Camunda BPM](https://camunda.com/de/products/) (Modeler und Process Engine)
- [GitHub](https://github.com/) (Versionierungstool)
- [Apache Tomcat](http://tomcat.apache.org/) (Web Server)
- [Genderize API](https://genderize.io/) (Überprüfung eines Nutzers)
- [Postman](https://www.getpostman.com/) (API Testing)
- [Visual Studio Code](https://code.visualstudio.com/) (Code Editor)
- [camunda-bpm-mail](https://github.com/camunda/camunda-bpm-mail) (Connector zur Versendung von E-Mails)

Des Weiteren wurden folgende Maven-Dependencies (`.jar-Dateien`) als Bibliothekerweiterungen verwendet:
- [Camunda BPM Connect Core](https://mvnrepository.com/artifact/org.camunda.connect/camunda-connect-core/1.0.3)
- [JavaMail API](https://mvnrepository.com/artifact/com.sun.mail/javax.mail/1.5.5)
- [SLF4J API Module](https://mvnrepository.com/artifact/org.slf4j/slf4j-api/1.7.21)
- [Camunda BPM Mail Core](https://mvnrepository.com/artifact/org.camunda.bpm.extension/camunda-bpm-mail-core/1.2.0)

## BPMN-Diagramm
Ein wichtiger Aspekt der im Projekt aufgenommen wurde, war die Terminverwaltung. Der Prozess “Terminerstellung mit externen Teilnehmern” wurde daher exemplarisch ausgewählt. Für die Modellierung dieses Prozesses wurde der [Camunda Modeler](https://camunda.com/de/products/modeler/) verwendet.

Zunächst soll ein Mitarbeiter des Unternehmens einen potentiellen Termin eingeben **(User-Task)**, welcher dann überprüft wird **(Business-Rule-Task)**. Ist der Termin verfügbar, kann der Teilnehmer von einem Mitarbeiter hinzugefügt werden **(User-Task)**. Im Falle, dass der Termin zeitlich nicht möglich ist, endet der Prozess. Nachdem der Teilnehmer hinzugefügt wurde, wird überprüft ob dieser im System vorhanden ist **(Service-Task)**. Ist er präsent, kann der Terminvorschlag versendet werden **(Send-Task)**. Andernfalls scheitert der Prozess und die Terminerstellung ist gescheitert.

![BPMN "Terminerstellung mit externen Teilnehmern”](/images/terminerstellung_extern_atomar.svg "BPMN-Diagramm Terminerstellung mit externen Teilnehmern")

## Technische Umsetzung
### User-Task - Termin eingeben
In der ersten User-Task gibt der Nutzer über ein Formular die nötigen Daten des gewünschten Termins ein. Dabei werden Inputs, welche für das DMN-Diagramm des nächsten Schrittes notwendig sind, erhoben. Diese sind **Wochentag** `Typ: long`, **Beginn** `Typ: long` und **Dauer_in_Minuten** `Typ: long`.

![Termin eingeben](/images/enter_date.PNG "Termin eingeben")

### Business-Rule-Task (DMN) - Termin prüfen
Der vorher eingegebene Termin wird mittels einer Business Rule Task ausgewertet. Falls die Daten valide sind und der Termin zur Verfügung steht, wird als Terminmöglichkeit `true` zurückgegeben. Im Falle, dass Konflikte mit Kernarbeitszeiten, Pausen oder regelmäßigen Terminen entstehen, ist der Termin nicht möglich und der Output ist `false`.

![Termin prüfen](/images/dmn_table.PNG "Termin prüfen")

Diese Outputs werden in einem XOR-Gateway ausgewertet, [siehe BPMN-Diagramm](#bpmn-diagramm). Wenn ein freier Termin zur Verfügung steht schreitet der Prozess fort, andernfalls scheitert er.

![Terminverfügbarkeit](/images/sequent_flow_1_positive.PNG "Termin verfügbar") ![Termin prüfen](/images/sequent_flow_1_negative.PNG "Termin nicht verfügbar")

In der DMN Tabelle sind Regeln für die Terminvergabe hinterlegt. Diese Regeln sollen sich am Unternehmensalltag orientieren. So ist es z.B. nur möglich Termine in der Kernarbeitszeit zu organisieren. Termine außerhalb der Kernarbeitszeit 8 Uhr - 18 Uhr oder in den allgemeinen Pausen (9-9.30 Uhr; 12-12.30 Uhr) werden nicht akzeptiert. Termine mit einer Dauer länger als 2 Stunden werden ebenfalls nicht akzeptiert.

Bei der Termineingabe sind folgende Termindaten erforderlich:

- Wochentag `int 1-7 (1 = Montag; 7 = Sontag)`
- Uhrzeit `Long z.B. 9.30 (für 9.30 Uhr)`
- Dauer `int z.B. 90 (90 Minuten)`

Werden diese Konventionen verletzt ist eine Prüfung nicht möglich.

![DMN Entscheidungstabelle](/images/dmn_tabelle_termin_dauer.svg "DMN Entscheidungstabelle")

### User-Task - Teilnehmer hinzufügen
Nun kann ein Teilnehmer zum Termin hinzugefügt werden. Hierbei werden wieder in einem Formular die Daten des Teilnehmers eingegeben, damit diese im darauffolgenden Schritt überprüft werden können. Der Name ist für diese Überprüfung essentiell und die E-Mail-Adresse wird für das spätere Ausstellen des Terminvorschlages benötigt.

![Teilnehmer hinzufügen](/images/add_participant.PNG "Teilnehmer hinzufügen")

### Service-Task - Teilnehmer in Datenbank überprüfen
Um das Vorhandensein des soeben hinzugefügten Teilnehmers in unserem System zu überprüfen, verwenden wir eine Service-Task. Diese Service-Task führt einen GET-Request an eine externe API mithilfe eines Connectors aus. Als Parameter sind hierbei die Methode `GET` bei GET-Requests, die URL `https://genderize.io/?name=${Name}` und die Headers `content-Type: application/json` nötig. Ist der Request erfolgreich, erhalten wir den Output in dem Parameter `Pruefung`. Dieses JSON-Response werten wir mithilfe von einem **Javascript-Inline-Script** aus, sodass nicht der gesamte Block, sondern nur ein Boolean-Wert zurückgegeben wird.
```javascript
var json = S(response);
var gender = json.prop("gender").value();

if (gender == "male" ) {
    var bool = true;
} else if (gender == "female") {
    var bool = true;
} else {
    var bool = false;
}

bool;
```
Das soll die Auswertung der Response im nächsten XOR-Gateway (“Name vorhanden?”) erleichtern. Ist der Teilnehmer schon im System vorhanden, kann der Terminvorschlag per E-Mail versendet werden. Kriegen wir als Reponse wiederum “null”, ist die Terminerstellung gescheitert und der Prozess endet.

![Name vorhanden](/images/sequent_flow_2_positive.PNG "Name vorhanden") ![Name nicht vorhanden](/images/sequent_flow_2_negative.PNG "Name nicht vorhanden")

### Send-Task - Termin versenden
An letzter Stelle können wir dem Teilnehmer im idealen Fall den Terminvorschlag per E-Mail senden. Dieser Prozess wird mithilfe eines **Mail-Send-Connectors** abgewickelt, der von [Camunda](https://github.com/camunda/camunda-bpm-mail) direkt angeboten wird. Dazu mussten wir die nötigen `.jar-Dateien` aus der Dokumentation entnehmen und zu den Bibliotheken hinzufügen. Außerdem war das Hinzufügen einer `mail-config.properties` Datei und eine Anpassung der Start-Datei zwingend erforderlich. Die nötigen Input-Parameter sind unter anderem ein **Adressat**, den wir aus dem vorherigen Formular entnehmen `${Mail}` aus [Teilnehmer hinzufügen](#user-task---teilnehmer-hinzufügen). Weiterhin werden ein Betreff `Typ: Text` und der Inhalt der E-Mail `Typ: Text` benötigt. Sind alle Anforderungen erfüllt, kann die E-Mail versendet werden und der Nutzer erhält die Nachricht mit seinem Terminvorschlag, wodurch der Prozess erfolgreich beendet wird.

![Termin versenden](/images/send_appointment.PNG "Termin versenden")

## Demo
Nach dem erfolgreichen Deployen des Prozesses, kann dieser über die Tasklist gestartet werden. Wenn man nun die Liste aktualisiert und zu `All tasks` navigiert, lässt sich ein Termin eingeben.

![Termin eingeben](/images/demo_form_1.PNG "Termin eingeben")

Um anzuzeigen, wie weit der Prozess fortgeschritten ist und ob die Eingaben richtig waren, kann dieser über das Cockpit inklusive der übertragenen Daten angezeigt werden.

![Teilnehmer hinzufügen](/images/demo_diagram_token.PNG "Teilnehmer hinzufügen")

![Termindaten](/images/demo_table.PNG "Termindaten")

Falls alles den eigenen Erwartungen entspricht, kann der Prozess wieder in der Tasklist weitergeführt werden. Der nächste Schritt beinhaltet das Hinzufügen eines Teilnehmers. Hierzu füllt man erneut ein Formular aus. Dabei ist zu beachten, dass möglichst reale Daten verwendet werden sollten, da ansonsten keine Benachrichtigung stattfinden kann.

![Teilnehmer hinzufügen](/images/demo_form_2.PNG "Teilnehmer hinzufügen")

Sind die Daten valide, folgt der Prozess weiterhin dem **Happy Path** und versendet eine E-Mail an den vorher angegebenen Teilnehmer. Herzlichen Glückwunsch, der Prozess wurde erfolgreich abgeschlossen!

![Versendete E-Mail](/images/demo_email.PNG "Versendete E-Mail")

___

_Diese Dokumentation ist Teil einer Projektarbeit im Rahmen der Veranstaltung “Auswahl und Anpassung von IT-Diensten” unter der Leitung von Frau Prof. Meister an der Technischen Hochschule Brandenburg._
