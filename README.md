# Anpassung des ausgewählten Kommunikationstools am Beispiel "Terminerstellung mit externen Teilnehmern"
Diese Dokumentation beschreibt die technische Implementierung einer Service-Orchestrierung des Prozesses “Terminerstellung mit externen Teilnehmern” mithilfe der Process Engine Camunda BPM. Aufgrund des Projektrahmens ist der Prozess “Terminerstellung mit externen Teilnehmern” Bestandteil eines fiktiven Unternehmens in Brandenburg an der Havel.

Ziel der folgenden Service-Orchestrierung ist die Umsetzung folgender Punkte:
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
- [camunda-bpm-mail](https://github.com/camunda/camunda-bpm-mail)(Connector zur Versendung von E-Mails)

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

### User-Task - Teilnehmer hinzufügen

Nun kann ein Teilnehmer zum Termin hinzugefügt werden. Hierbei werden wieder in einem Formular die Daten des Teilnehmers eingegeben, damit diese im darauffolgenden Schritt überprüft werden können. Der Name ist für diese Überprüfung essentiell und die E-Mail-Adresse wird für das spätere Ausstellen des Terminvorschlages benötigt.

![Teilnehmer hinzufügen](/images/add_participant.PNG "Teilnehmer hinzufügen")
