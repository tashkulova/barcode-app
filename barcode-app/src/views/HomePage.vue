<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Barcode Scanner App</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">
      <!-- Button für Kamera-Scan -->
      <ion-button expand="block" @click="scanBarcode">
        <ion-icon slot="start" :icon="scanOutline"></ion-icon>
        Code zum Scannen finden
      </ion-button>

      <!-- Liste der Barcodes -->
      <ion-list>
        <ion-item-sliding v-for="barcode in barcodes" :key="barcode.id">
          <ion-item>
            <ion-label>
              <h2>{{ barcode.displayValue }}</h2>
              <p>Format: {{ barcode.format }} | Typ: {{ barcode.valueType }}</p>
            </ion-label>
            <!-- Aktionen für URL/PHONE -->
            <ion-button v-if="['URL', 'PHONE'].includes(barcode.valueType)" slot="end" fill="outline" @click="handleAction(barcode)">
              {{ barcode.valueType === 'URL' ? 'Browser' : 'Anruf' }}
            </ion-button>
          </ion-item>

          <ion-item-options side="end">
            <ion-item-option color="primary" @click="copyToClipboard(barcode)">Kopieren</ion-item-option>
            <ion-item-option color="danger" @click="removeBarcode(barcode.id)">Löschen</ion-item-option>
          </ion-item-options>
        </ion-item-sliding>
      </ion-list>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton, IonIcon, IonList, IonItem, IonLabel, IonItemSliding, IonItemOptions, IonItemOption } from '@ionic/vue';
import { scanOutline } from 'ionicons/icons';
import { BarcodeScanner } from '@capacitor-mlkit/barcode-scanning';
import { Preferences } from '@capacitor/preferences';
import { Clipboard } from '@capacitor/clipboard';
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
    async scanBarcode() {
      const { barcodes } = await BarcodeScanner.scan();
      if (barcodes.length > 0) {
        const newEntry = {
          id: Date.now().toString(),
          displayValue: barcodes[0].displayValue,
          format: barcodes[0].format,
          valueType: barcodes[0].valueType // URL, PHONE, etc.
        };
        this.barcodes.unshift(newEntry);
        await Preferences.set({ key: 'barcodes', value: JSON.stringify(this.barcodes) });
      }
    },
    async handleAction(barcode: any) {
      if (barcode.valueType === 'URL') {
        await Browser.open({ url: barcode.displayValue });
      } else if (barcode.valueType === 'PHONE') {
        window.open(`tel:${barcode.displayValue}`);
      }
    },
    async copyToClipboard(barcode: any) {
      await Clipboard.write({ string: barcode.displayValue });
    },
    async removeBarcode(id: string) {
      this.barcodes = this.barcodes.filter(b => b.id !== id);
      await Preferences.set({ key: 'barcodes', value: JSON.stringify(this.barcodes) });
    }
  }
});
</script>