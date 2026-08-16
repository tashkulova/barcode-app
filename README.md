# Barcode Scanner App

Eine hybride mobile Anwendung für Android, entwickelt mit **Ionic Framework**, **Vue.js** und **Capacitor**. Die App ermöglicht das Scannen, Verwalten und Nutzen von Barcodes (URL/PHONE).

## Funktionen
*   **Scannen:** Unterstützung für Kamera-Scan und Import von Bildern aus der Galerie.
*   **Persistenz:** Gescannte Barcodes werden lokal gespeichert und bleiben nach App-Neustart erhalten.
*   **Aktionen:** 
    *   **URL-Typ:** Öffnet Webseiten direkt im In-App-Browser.
    *   **PHONE-Typ:** Startet Anrufe über die Standard-Telefon-App.
*   **Verwaltung:** Barcodes können kopiert, geteilt oder aus der Liste gelöscht werden (Wischgeste nach links).
*   **Listenansicht:** Übersichtliche Darstellung von Wert, Format und Typ jedes Barcodes.

## Voraussetzungen
*   **Hardware:** Physisches Android-Gerät mit aktiven Google Play Services.
*   **Berechtigungen:** Die App benötigt Zugriff auf die Kamera und den Speicher für Galerie-Bilder.

## Technische Umsetzung
Die App basiert auf modernem Web-Stack und nutzt native Capacitor-Plugins:
*   **Barcode-Erkennung:** `@capacitor-mlkit/barcode-scanning`
*   **Kamera & Galerie:** `@capacitor/camera`
*   **Speicherung:** `@capacitor/preferences`
*   **UI/Framework:** `Ionic Framework` mit `Vue.js`

### Lokale Entwicklung (via GitHub Codespaces):
1.  Repository öffnen.
2.  In den Projektordner wechseln: `cd barcode-app`
3.  Abhängigkeiten installieren: `npm install`
4.  App bauen: `npm run build`
5.  Capacitor synchronisieren: `npx cap sync`


## Installation auf dem Smartphone
1.  Übertrage die `app-debug.apk` auf dein Android-Gerät.
2.  Tippe auf die Datei, um die Installation zu starten.
3.  *Hinweis:* Aktiviere ggf. in den Android-Einstellungen die "Installation aus unbekannten Quellen".
4.  Beim ersten Start die Berechtigungen für Kamera und Speicher bestätigen.

## Verwendete KI-Tools & Dokumentation
* **Genutztes Tool:** Google Gemini (KI-Assistent)
* **Einsatzbereich & Zweck:** 
  * Hilfe beim Debugging von Code-Abschnitten im Bereich des Capacitor-Plugins und der UI-Steuerung via Vue.js.
  * Unterstützung bei der Strukturierung und Ausarbeitung der Präsentationsinhalte.

## Lizenz
Dieses Projekt wurde im Rahmen des DHBW-Moduls "Fallstudie" erstellt.