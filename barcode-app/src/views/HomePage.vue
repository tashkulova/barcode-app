<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Barcode Scanner</ion-title>
      </ion-toolbar>
    </ion-header>
    
    <ion-content class="ion-padding">
      <!-- Listenansicht der gescannten Barcodes -->
      <ion-list v-if="barcodes.length > 0">
        <ion-item-sliding v-for="barcode in barcodes" :key="barcode.id">
          <ion-item>
            <ion-label>
              <h2>{{ barcode.displayValue }}</h2>
              <p>Format: {{ barcode.format }} | Typ: {{ barcode.valueType }}</p>
            </ion-label>
            <!-- Button zum Öffnen (nur URL oder PHONE) -->
            <ion-button 
              v-if="barcode.valueType === 'URL' || barcode.valueType === 'PHONE'" 
              slot="end" 
              @click="openBarcode(barcode)"
            >
              Öffnen
            </ion-button>
          </ion-item>

          <!-- Interaktionsoptionen (Teilen, Kopieren, Löschen) -->
          <ion-item-options side="end">
            <ion-item-option color="primary" @click="copyBarcode(barcode)">Kopie</ion-item-option>
            <ion-item-option color="tertiary" @click="shareBarcode(barcode)">Teilen</ion-item-option>
            <ion-item-option color="danger" @click="deleteBarcode(barcode.id)">Löschen</ion-item-option>
          </ion-item-options>
        </ion-item-sliding>
      </ion-list>
      
      <div v-else class="ion-text-center ion-margin-top">
        <p>Keine Barcodes vorhanden. Starte einen Scan!</p>
      </div>

      <!-- Floating Action Button für den Scan -->
      <ion-fab vertical="bottom" horizontal="end" slot="fixed">
        <ion-fab-button @click="presentScanOptions">
          <ion-icon :icon="scanOutline"></ion-icon>
        </ion-fab-button>
      </ion-fab>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { 
  IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonList, 
  IonItem, IonLabel, IonButton, IonFab, IonFabButton, IonIcon, 
  IonItemSliding, IonItemOptions, IonItemOption, actionSheetController 
} from '@ionic/vue';
import { scanOutline } from 'ionicons/icons';
import { BarcodeScanner } from '@capacitor-mlkit/barcode-scanning';
import { Camera, CameraResultType, CameraSource } from '@capacitor/camera';
import { Preferences } from '@capacitor/preferences';
import { Clipboard } from '@capacitor/clipboard';
import { Share } from '@capacitor/share';
import { Browser } from '@capacitor/browser';

interface ScannedBarcode {
  id: string;
  displayValue: string;
  format: string;
  valueType: string;
}

export default defineComponent({
  name: 'HomePage',
  components: {
    IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonList,
    IonItem, IonLabel, IonButton, IonFab, IonFabButton, IonIcon,
    IonItemSliding, IonItemOptions, IonItemOption
  },
  data() {
    return {
      scanOutline,
      barcodes: [] as ScannedBarcode[]
    };
  },
  async mounted() {
    await this.loadBarcodes();
  },
  methods: {
    // 1. Persistenz: Barcodes aus dem Speicher laden
    async loadBarcodes() {
      const { value } = await Preferences.get({ key: 'saved_barcodes' });
      if (value) {
        this.barcodes = JSON.parse(value);
      }
    },
    // 2. Persistenz: Barcodes im Speicher sichern
    async saveBarcodes() {
      await Preferences.set({
        key: 'saved_barcodes',
        value: JSON.stringify(this.barcodes)
      });
    },
    // 3. Auswahlmenü: Kamera oder Galerie
    async presentScanOptions() {
      const actionSheet = await actionSheetController.create({
        header: 'Barcode scannen',
        buttons: [
          { text: 'Kamera verwenden', handler: () => { this.scanWithCamera(); } },
          { text: 'Aus Galerie wählen', handler: () => { this.scanFromGallery(); } },
          { text: 'Abbrechen', role: 'cancel' }
        ]
      });
      await actionSheet.present();
    },
    // 4. Scannen per Kamera
    async scanWithCamera() {
      // Berechtigungen anfordern (zwingend nötig)
      await BarcodeScanner.requestPermissions();
      document.querySelector('body')?.classList.add('barcode-scanner-active');
      
      try {
        const { barcodes } = await BarcodeScanner.scan();
        if (barcodes.length > 0) {
          this.addBarcodeToList(barcodes[0]);
        }
      } catch (error) {
        console.error(error);
      } finally {
        document.querySelector('body')?.classList.remove('barcode-scanner-active');
      }
    },
    // 5. Scannen per Bild aus der Galerie
    async scanFromGallery() {
      try {
        const image = await Camera.getPhoto({
          quality: 100,
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
        console.error("Fehler beim Lesen aus Galerie: ", error);
      }
    },
    // 6. Neuen Barcode zur Liste hinzufügen und persistieren
    addBarcodeToList(barcode: any) {
      const newBarcode: ScannedBarcode = {
        id: new Date().getTime().toString(),
        displayValue: barcode.displayValue,
        format: barcode.format,
        valueType: barcode.valueType
      };
      this.barcodes.unshift(newBarcode);
      this.saveBarcodes();
    },
    // 7. Löschen von Barcodes
    deleteBarcode(id: string) {
      this.barcodes = this.barcodes.filter(b => b.id !== id);
      this.saveBarcodes();
    },
    // 8. In Zwischenablage kopieren
    async copyBarcode(barcode: ScannedBarcode) {
      await Clipboard.write({
        string: barcode.displayValue
      });
      alert('In die Zwischenablage kopiert!');
    },
    // 9. Barcode Teilen
    async shareBarcode(barcode: ScannedBarcode) {
      await Share.share({
        title: 'Gescanntes Ergebnis',
        text: barcode.displayValue,
        dialogTitle: 'Barcode teilen'
      });
    },
    // 10. URLs oder Telefonnummern öffnen
    async openBarcode(barcode: ScannedBarcode) {
      if (barcode.valueType === 'URL') {
        // Öffnen im In-App-Browser
        await Browser.open({ url: barcode.displayValue });
      } else if (barcode.valueType === 'PHONE') {
        // Als Anruf über die Standard-Telefon-App öffnen
        window.open('tel:' + barcode.displayValue);
      }
    }
  }
});
</script>

<style>
/* Versteckt den HTML-Hintergrund, damit das Kamera-Overlay sichtbar wird */
body.barcode-scanner-active {
  background: transparent;
}
</style>