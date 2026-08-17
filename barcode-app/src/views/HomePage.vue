<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Barcode Scanner-App</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <!-- Button über den ein neuer Barcode per Kamera oder Galerie gescannt werden kann -->
      <ion-button expand="block" @click="presentScanOptions" class="ion-margin-bottom primary-btn">
        <ion-icon slot="start" :icon="scanOutline"></ion-icon>
        Neuen Barcode scannen
      </ion-button>

      <!-- Listenansicht, die alle gescannten Barcodes inklusive Wert (displayValue), Format und Werttyp auflistet -->
      <ion-list v-if="barcodes.length > 0">
        <ion-item-sliding v-for="barcode in barcodes" :key="barcode.id">
          <ion-item>
            <ion-label class="ion-text-wrap">
              <h2><strong>Wert:</strong> {{ barcode.displayValue }}</h2>
              <p><strong>Format:</strong> {{ barcode.format }} | <strong>Typ:</strong> {{ barcode.valueType }}</p>
            </ion-label>

            <!-- Button zum Öffnen von URLs im In-App-Browser oder PHONE als Anruf -->
            <ion-button 
              v-if="barcode.valueType === 'URL' || barcode.valueType === 'PHONE'" 
              slot="end" 
              fill="solid" 
              size="small"
              class="secondary-btn"
              @click="openBarcode(barcode)"
            >
              {{ barcode.valueType === 'URL' ? 'Im Browser öffnen' : 'Anrufen' }}
            </ion-button>
          </ion-item>

          <!-- Optionen zum Kopieren, Teilen und Löschen über die Listenansicht -->
          <ion-item-options side="end">
            <ion-item-option color="secondary-btn" @click="copyBarcode(barcode)">Kopieren</ion-item-option>
            <ion-item-option color="secondary-btn" @click="shareBarcode(barcode)">Teilen</ion-item-option>
            <ion-item-option color="danger" @click="deleteBarcode(barcode.id)">Löschen</ion-item-option>
          </ion-item-options>
        </ion-item-sliding>
      </ion-list>

      <div v-else class="ion-text-center ion-margin-top">
        <p>Noch keine Barcodes gescannt. Starte den Scanner über den Button oben.</p>
      </div>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { 
  IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton, 
  IonIcon, IonList, IonItem, IonLabel, IonItemSliding, IonItemOptions, 
  IonItemOption, actionSheetController 
} from '@ionic/vue';
import { scanOutline } from 'ionicons/icons';
import { BarcodeScanner } from '@capacitor-mlkit/barcode-scanning';
import { Camera, CameraResultType, CameraSource } from '@capacitor/camera';
import { Preferences } from '@capacitor/preferences';
import { Clipboard } from '@capacitor/clipboard';
import { Share } from '@capacitor/share';
import { Browser } from '@capacitor/browser';

interface BarcodeItem {
  id: string;
  displayValue: string;
  format: string;
  valueType: string;
}

export default defineComponent({
  name: 'HomePage',
  components: {
    IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton,
    IonIcon, IonList, IonItem, IonLabel, IonItemSliding, IonItemOptions,
    IonItemOption
  },
  data() {
    return {
      scanOutline,
      barcodes: [] as BarcodeItem[]
    };
  },
  async mounted() {
    // Beim Start der App gespeicherte Barcodes laden
    await this.loadBarcodes();
  },
  methods: {
    async loadBarcodes() {
      const { value } = await Preferences.get({ key: 'saved_barcodes' });
      if (value) {
        this.barcodes = JSON.parse(value);
      }
    },
    async saveBarcodes() {
      await Preferences.set({
        key: 'saved_barcodes',
        value: JSON.stringify(this.barcodes)
      });
    },

    // Auswahlmenü: Entweder Kamera oder Bild aus Galerie wählen
    async presentScanOptions() {
      const actionSheet = await actionSheetController.create({
        header: 'Barcode-Quelle wählen',
        buttons: [
          {
            text: 'Mit Kamera scannen',
            handler: () => {
              this.scanWithCamera();
            }
          },
          {
            text: 'Aus Galerie auswählen',
            handler: () => {
              this.scanFromGallery();
            }
          },
          {
            text: 'Abbrechen',
            role: 'cancel'
          }
        ]
      });
      await actionSheet.present();
    },
   
    // 1. Kamera-Scan inklusive automatischer Berechtigungsanforderung zur Laufzeit
    async scanWithCamera() {
      try {
        await BarcodeScanner.requestPermissions();
        const { barcodes } = await BarcodeScanner.scan();
        if (barcodes.length > 0) {
          this.addBarcodeToList(barcodes[0]);
        }
      } catch (error) {
        console.error('Kamera-Scan fehlgeschlagen:', error);
      }
    },
   
    // 2. Galerie-Scan über den File Picker
    async scanFromGallery() {
      try {
        const image = await Camera.getPhoto({
          quality: 90,
          allowEditing: false,
          resultType: CameraResultType.Uri,
          source: CameraSource.Photos
        });

        if (image.path) {
          const { barcodes } = await BarcodeScanner.readBarcodesFromImage({
            path: image.path,
          });
          if (barcodes.length > 0) {
            this.addBarcodeToList(barcodes[0]);
          }
        }
      } catch (error) {
        console.error('Galerie-Scan fehlgeschlagen:', error);
      }
    },
   
    // Fügt den gescannten Barcode direkt der Liste hinzu, persistiert ihn und zeigt ihn sofort an
    addBarcodeToList(rawBarcode: any) {
      const newBarcode: BarcodeItem = {
        id: Date.now().toString(),
        displayValue: rawBarcode.displayValue || 'Unbekannter Wert',
        format: rawBarcode.format || 'Unbekannte Format',
        valueType: rawBarcode.valueType || 'TEXT'
      };
      this.barcodes.unshift(newBarcode);
      this.saveBarcodes();
    },
   
    // Löschen eines einzelnen Barcodes über die Listenansicht
    async deleteBarcode(id: string) {
      this.barcodes = this.barcodes.filter(b => b.id !== id);
      await this.saveBarcodes();
    },
   
    // In Zwischenablage kopieren
    async copyBarcode(barcode: BarcodeItem) {
      await Clipboard.write({ string: barcode.displayValue });
      alert('Erfolgreich in die Zwischenablage kopiert!');
    },
   
    // Barcode über die Listenansicht teilen
    async shareBarcode(barcode: BarcodeItem) {
      await Share.share({
        title: 'Gescannter Barcode',
        text: barcode.displayValue,
        dialogTitle: 'Barcode teilen'
      });
    },
    
     // Methode für das Öffnen von URL und PHONE
    async openBarcode(barcode: any) {
      try {
        if (barcode.valueType === 'URL') {
          let url = barcode.displayValue;
           // Stellt sicher, dass immer http/https davorsteht, sonst streikt das Browser-Plugin
          if (!url.startsWith('http://') && !url.startsWith('https://')) {
            url = 'https://' + url;
          }
           await Browser.open({ url: url });
          
        } else if (barcode.valueType === 'PHONE') {
          let rawValue = barcode.displayValue;
            let phone = '';
            if (rawValue.startsWith('tel:')) {
                // Fall 1: 'tel:' ist bereits vorhanden
                phone = rawValue.replace('tel:', '');
                window.open('tel:' + phone, '_system');
            } else if (rawValue.startsWith('+49')) {
                // Fall 2: Kein 'tel:', aber beginnt mit '+49'
                phone = rawValue;
                window.open('tel:' + phone, '_system');
            }
        }
      } catch (error) {
        console.error("Fehler beim Öffnen: ", error);
        alert("Aktion konnte nicht ausgeführt werden.");
      }
    }
  }
   
});
</script>