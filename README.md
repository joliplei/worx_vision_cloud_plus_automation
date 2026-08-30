<p align="center">
  <img src="assets/worx-vision-cloud-plus-automation.png" alt="Worx Vision Cloud PLUS - Smart Mowing Automation for Home Assistant" width="100%">
</p>

Intelligente Mäh-Automatisierung für Worx-Mäher in Home Assistant, fokussiert auf Worx Vision / RTK Modelle und weiterhin kompatibel mit älteren kabelgebundenen Worx-Mähern.

Dieser Blueprint ersetzt einen starren Mähzeitplan durch einen wetterabhängigen Plan. Er schätzt das Graswachstum, wählt ein trockenes und sinnvolles Mähfenster, startet zuerst den Kantenschnitt und beginnt dann mit dem normalen Mähen, nachdem der Mäher zur Ladestation zurückgekehrt ist und aufgeladen wurde. Vision / RTK Mäher werden durch automatische Lade- und Fortsetzungszyklen überwacht, bis sie wirklich fertig sind; ältere kabelgebundene Worx-Mäher können weiterhin eine berechnete Laufzeit verwenden.

Dieses Repository ist absichtlich vom Repository der benutzerdefinierten Integration getrennt:

- Integrationscode befindet sich in [`worx_vision_cloud_plus_github`](https://github.com/joliplei/worx_vision_cloud_plus_github),
- Automatisierungen und Blueprints befinden sich hier.

Ursprünglich erstellt von **Smart Service**. Übersetzt ins Deutsche von **joliplei**.

## Blueprint importieren

[![Open your Home Assistant instance and import this blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fjoliplei%2Fworx_vision_cloud_plus_automation%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fworx_vision_cloud_plus%2Fsmart_mowing_schedule.yaml)

Manuelle Import-URL:

```text
https://github.com/joliplei/worx_vision_cloud_plus_automation/blob/main/blueprints/automation/worx_vision_cloud_plus/smart_mowing_schedule.yaml
```

## Was es macht

- Schätzt das Graswachstum einmal täglich mit einem Growth Potential (GP) Modell.
- Nutzt lokalen Regen, Temperatur, Sonneneinstrahlung/UV, optionale Außenluftfeuchtigkeit und Bodenfeuchtigkeit, sowie Bewässerungs- und Düngungseinstellungen.
- Akzeptiert eine separate optionale `weather` Entität für die stündliche Planung.
- Wählt die beste Mähzeit im gewählten Zeitfenster, vermeidet Regen, nasses Gras und unsichere Temperaturen in den nächsten drei Stunden.
- Führt zuerst den Kantenschnitt durch, wartet, bis der Mäher zur Ladestation zurückkehrt und auf mindestens `80%` aufgeladen ist, und startet dann das normale einmalige Mähen.
- Lässt den Benutzer den Mähzyklus-Typ wählen: Vision / RTK selbstbeendendes Mähen oder zeitgesteuertes Mähen für ältere kabelgebundene Worx-Mäher.
- Hält drei Helfer aktuell: geschätztes Graswachstum, letztes vollständiges Mähen und nächstes geplantes Mähen.
- Erkennt manuelles Mähen aus der WORX-App und setzt die Wachstumsschätzung erst zurück, nachdem der vollständige Mähzyklus bestätigt wurde.
- Überprüft Live-Messungen, Worx Cloud Regenverzögerung und die stündliche Vorhersage erneut, bevor das normale Mähen nach dem Kantenschnitt beginnt.

## Mählogik

Der automatische Startschwellenwert ist für das Roboter-Mähen optimiert. Anstatt auf hohes Gras zu warten, bevorzugt der Blueprint häufige, leichte Schnitte, normalerweise bei einem geschätzten Wachstum von `2-4.5 mm`, abhängig von der gewählten Schnitthöhe. Die Ein-Drittel-Blatt-Regel wird als Sicherheitsgrenze beibehalten, nicht als normales Ziel.

Im automatischen Modus führt der Blueprint eine Vorhersageberechnung nach dem täglichen Graswachstums-Update durch und speichert eine konkrete Mähzeit. Das gewählte Zeitfenster darf keinen Regen in der Vorhersage haben und muss Temperatur und Luftfeuchtigkeit während des erwarteten Mähhorizonts im sicheren Bereich halten. Diese gespeicherte Zeit wird durch Hintergrund-Vorhersageänderungen nicht verschoben. Standardmäßig ist das Mähen nur zwischen `10 °C` und `25 °C` erlaubt. Wenn es zu heiß ist, sucht der Blueprint zuerst nach der nächsten kühleren Zeit am selben Tag. Ohne FiatLux sucht er bis `22:00`; mit bestätigtem Zubehör kann er die Suche über Nacht bis `05:00` fortsetzen.

Für die besten lokalen Entscheidungen verwende deine eigene Wetterstation oder lokale Außensensoren für die gemessenen Bedingungen. **MeteoFusion HA** ist die empfohlene stündliche Vorhersagequelle: Der Blueprint erkennt automatisch dessen `smart_service.weather.v1` Kontext, verwendet die Live-Regenrate und den täglichen Niederschlag und bevorzugt Vorhersage-Zeitfenster mit höherer MeteoFusion-Sicherheit. Eine Standard `weather` Entität wie Tomorrow.io bleibt unterstützt. Wenn keine stündliche Vorhersage ausgewählt ist, verwendet der Blueprint das konfigurierte Mähfenster und validiert die Live-Bedingungen zum Startzeitpunkt.

Der vorhergesagte Regen wird bei der Auswahl der ursprünglichen Mähzeit berücksichtigt, sodass der gespeicherte Plan ein wirklich trockenes Vorhersagefenster bevorzugt. Zur gespeicherten Startzeit wird ein funktionierender binärer Regensensor maßgeblich: ein trockener Sensor ermöglicht den Start des Zyklus, selbst wenn sich die Vorhersage nach der Planung geändert hat. Wenn der Sensor während des Kanten- oder normalen Mähens auf Regen wechselt, wird der Roboter zur Ladestation zurückgeschickt. Wenn der ausgewählte Regensensor nicht verfügbar ist, bleiben MeteoFusion oder die Standard-Wetterentität der schützende Fallback.

Die Trocknungsverzögerung nach Regen beginnt nur nach einem echten `on` zu `off` Wechsel des ausgewählten Regensensors. Ein Neustart von Home Assistant oder ein vorübergehender `unavailable` Status gefolgt von `off` kann den Trocknungs-Timer nicht neu starten. Spurenniederschlag unter `0.2 mm` wird ignoriert, was verhindert, dass Sensorrauschen das Mähen wiederholt verschiebt.

Lange stündliche Vorhersagen werden auf den Planungshorizont begrenzt, sodass Anbieter wie Pirate Weather viele Vorhersagedatensätze zurückgeben können, ohne dass die Home Assistant Vorlage ihr Ausgabelimit überschreitet.

Wenn tatsächlicher Regen oder eine unsichere gemessene Temperatur den gespeicherten Start blockiert, führt der Blueprint eine neue Vorhersageberechnung durch, speichert eine neue konkrete Zeit und erklärt den Grund in der Benachrichtigung. Benachrichtigungen sind auf die tägliche Graswachstumsberechnung, eine echte Verschiebung beim Start, den Mähstart und den Mäh-Abschluss oder -Abbruch beschränkt.

## Bevor du startest

Deaktiviere den Mähzeitplan in der WORX-App, damit Home Assistant der einzige Planer ist, der die Mäher-Starts steuert.

Erstelle drei Home Assistant Helfer, bevor du die Automatisierung erstellst:

- `input_number` für das geschätzte Graswachstum, zum Beispiel `input_number.worx_estimated_grass_growth_mm`.
- `input_datetime` für den letzten vollständigen Mähzyklus, zum Beispiel `input_datetime.worx_last_full_mow`.
- `input_datetime` für die nächste geplante Mähzeit, zum Beispiel `input_datetime.worx_next_planned_mow`.

Dokumentation:

- [Smart Mowing Einrichtung](docs/smart-mowing-schedule.md)
- [Beispiel für ein Helper-Paket](docs/smart-mowing-helpers-package.yaml)

Blueprint-Pfad:

```text
blueprints/automation/worx_vision_cloud_plus/smart_mowing_schedule.yaml
```

## Anforderungen

- Home Assistant 2025.1.0 oder neuer.
- Worx Vision Cloud PLUS Integration `1.3.1` oder neuer, installiert von [`joliplei/worx_vision_cloud_plus_github`](https://github.com/joliplei/worx_vision_cloud_plus_github).
- Eine `lawn_mower` Entität für den Mäher.
- Der Dienst für einmaliges Mähen aus der Worx Vision Cloud PLUS Integration.
- Batterie- und Regen-Entitäten aus der Integration.
- Temperatur-, Regen-/Wetter- und Sonneneinstrahlung/UV-Quellen aus Home Assistant.
- Optionale Einstellung für automatische Bewässerung für Rasenflächen, die regelmäßig bewässert werden.
- Drei Helfer: geschätztes Graswachstum, letztes vollständiges Mähen, nächstes geplantes Mähen.
- Optionaler Bodenfeuchtigkeitssensor.

## Unterstützung

Wenn dir dieses Projekt hilft, kannst du den ursprünglichen Entwickler Smart Service unterstützen:

[Donate via Revolut](https://revolut.me/smartserwis)

## Datenschutz

Veröffentliche keine Home Assistant Speicherdateien, Zugriffstokens, Seriennummern, rohe API-Antworten oder Screenshots, die genaue Gartenkoordinaten zeigen.
