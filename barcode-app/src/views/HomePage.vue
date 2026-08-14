<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Barcode Scanner App</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <ion-button expand="block" @click="presentScanOptions" class="ion-margin-bottom">
        <ion-icon slot="start" :icon="scanOutline"></ion-icon>
        Neuen Barcode scannen
      </ion-button>

      <ion-list v-if="barcodes.length > 0">
        <ion-item-sliding v-for="barcode in barcodes" :key="barcode.id">
          <ion-item>
            <ion-label class="ion-text-wrap">
              <h2>{{ barcode.productName }}</h2>
              <p>
                <strong>Typ:</strong> {{ barcode.valueType }} | <strong>Format:</strong> {{ barcode.format }}<br>
                <strong>Nummer:</strong> {{ barcode.displayValue }}<br>
                <strong>Hersteller:</strong> {{ barcode.manufacturer }} | <strong>Land:</strong> {{ barcode.country }}
              </p>
            </ion-label>
            <!-- Hinzugefügter Button zum Öffnen von URLs oder Anrufen bei PHONE -->
            <ion-button 
              v-if="barcode.valueType === 'URL' || barcode.valueType === 'PHONE'" 
              slot="end" 
              size="small"
              @click="openBarcode(barcode)"
            >
              {{ barcode.valueType === 'URL' ? 'Browser' : 'Anrufen' }}
            </ion-button>
          </ion-item>

          <ion-item-options side="end">
            <ion-item-option color="danger" @click="removeBarcode(barcode.id)">Löschen</ion-item-option>
          </ion-item-options>
        </ion-item-sliding>
      </ion-list>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton, IonIcon, IonList, IonItem, IonLabel, IonItemSliding, IonItemOptions, IonItemOption, actionSheetController } from '@ionic/vue';
import { scanOutline } from 'ionicons/icons';
import { BarcodeScanner } from '@capacitor-mlkit/barcode-scanning';
import { Preferences } from '@capacitor/preferences';
import { Browser } from '@capacitor/browser';

export default defineComponent({
  components: { IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton, IonIcon, IonList, IonItem, IonLabel, IonItemSliding, IonItemOptions, IonItemOption },
  data() {
    return { scanOutline, barcodes: [] as any[] };
  },
  async mounted() {
    const { value } = await Preferences.get({ key: 'barcodes' });
    if (value) this.barcodes = JSON.parse(value);
  },
  methods: {
    // Hilfsfunktion zur Analyse der EAN-13 Nummer
    parseBarcodeDetails(value: string) {
      const countryCodes: { [key: string]: string } = {
        '40': 'Deutschland', '45': 'Japan', '49': 'Japan', '50': 'Großbritannien', '76': 'Schweiz'
      };
      
      const country = countryCodes[value.substring(0, 2)] || 'Unbekannt';
      const manufacturer = value.substring(2, 7); // Beispielhafte Extraktion
      
      return {
        productName: "Produkt #" + value.substring(7, 12),
        country: country,
        manufacturer: manufacturer
      };
    },

    async scanBarcode() {
      const { barcodes } = await BarcodeScanner.scan();
      if (barcodes.length > 0) {
        const raw = barcodes[0];
        const details = this.parseBarcodeDetails(raw.displayValue);
        
        const newEntry = {
          id: Date.now().toString(),
          displayValue: raw.displayValue,
          format: raw.format,
          valueType: raw.valueType,
          ...details
        };
        
        this.barcodes.unshift(newEntry);
        await Preferences.set({ key: 'barcodes', value: JSON.stringify(this.barcodes) });
      }
    },
    
    async presentScanOptions() {
      // Vereinfacht für Kamera-Start
      this.scanBarcode();
    },

    async removeBarcode(id: string) {
      this.barcodes = this.barcodes.filter(b => b.id !== id);
      await Preferences.set({ key: 'barcodes', value: JSON.stringify(this.barcodes) });
    },

    // Hinzugefügte Methode für das Öffnen von URL und PHONE
    async openBarcode(barcode: any) {
      if (barcode.valueType === 'URL') {
        await Browser.open({ url: barcode.displayValue });
      } else if (barcode.valueType === 'PHONE') {
        window.location.href = 'tel:' + barcode.displayValue;
      }
    }
  }
});
</script>