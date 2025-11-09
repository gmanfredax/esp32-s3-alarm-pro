# Centrale Allarme ESP32-S3 + W5500

Firmware per centrale di allarme basata su:

- MCU: ESP32-S3-WROOM-1-N16R8 / ESP32-S3-WROOM-2-N32R16V  
- Ethernet: W5500 (SPI dedicata)  
- NFC: PN532 (SPI dedicata)  
- Bus CAN (TWAI) per espansioni  
- Expander GPIO: MCP23017 (I²C)  
- 10 zone analogiche 0–12 V (EOL / 2EOL / 3EOL)  
- Misura BUS 13,8 V (batteria/alimentatore)  
- USB nativa (CDC) per debug/programmazione  

Il progetto è sviluppato con ESP-IDF + FreeRTOS, architettura modulare e orientata agli eventi.

---

## Funzionalità principali

- **Gestione zone locali (Z1..Z10)**  
  - 10 ingressi ADC dedicati.  
  - Sampling 100 Hz, filtro media mobile + mediana.  
  - Modalità: NONE/NC/NO, EOL_1R, EOL_2R_TAMPER, EOL_3R_TAMPER.  
  - Stati logici: RIPOSO / ALLARME / TAMPER_ZONA / GUASTO, con debounce.  

- **Tamper globale**  
  - Ingresso digitale `TAMPER_GLOBAL` (MCP23017 B6).  
  - Loop NC in serie su tamper sensori/sirene/centrale.  

- **Monitor BUS 13,8 V / batteria**  
  - Lettura ADC su VBUS (GPIO14, partitore 47k/10k).  
  - Soglie configurabili: OK / BATTERIA BASSA / CRITICA.  

- **Ethernet W5500**  
  - SPI2 dedicata (SCLK/MOSI/MISO/CS/INT/RST).  
  - Integrazione con `esp-netif` / LwIP.  
  - DHCP client, gestione link up/down.

- **Provisioning HTTP + Claim MQTT**  
  - Se il pannello non è provisionato:  
    - HTTP server su W5500, form di configurazione:
      - MQTT host/port, CA PEM, claim code.  
    - Salvataggio in NVS.  
  - Claim verso EMQX:
    - Connessione MQTTS con utente `nsbootstrap`.  
    - PUB: `provision/claim/<client_id>`  
    - SUB: `provision/reply/<client_id>`  
    - Ricezione `username` / `password` definitivi device, salvataggio in NVS.  
  - Dopo il claim:
    - Disattivazione HTTP.  
    - Connessione permanente come utente device:  
      - Namespace: `nsalarmpro/<device_id>/...`  
      - Heartbeat periodico su `status`.  
      - SUB comandi su `cmd/#`.

- **MCP23017 – LED e uscite**  
  - Porta A: LED (ALLARME, MANUTENZIONE, RGB provisioning, FAULT...).  
  - Porta B: DOUT1..5, `SERVICE_IN`, `TAMPER_GLOBAL`.  
  - API alto livello per LED/pattern, uscite e ingressi lenti.

- **LED locali**  
  - `LED_STATO` su GPIO0: disinserito / inserito / anomalia (blink lento).  
  - LED ALLARME / MANUTENZIONE su MCP23017.  
  - LED RGB provisioning (A2..A4) per stato provisioning/claim.

- **NFC PN532**  
  - SPI3 dedicata, RST su TXD0, IRQ su RXD0.  
  - Scan continuo card, lettura UID, distinzione TAG UTENTE / TAG MANUTENZIONE (tabella demo).  

- **Bus CAN (TWAI)**  
  - TWAI su GPIO38/39.  
  - Stub per auto-discovery moduli di espansione (zone/uscite remoti) e diagnostica bus.

- **Event log centralizzato**  
  - Modulo `event_log` con buffer circolare di eventi.  
  - Categorie: zone, tamper, BUS, NFC, CAN, MQTT/claim, sistema, ecc.  
  - Ogni evento: timestamp, tipo, severità, sorgente, messaggio corto.  
  - Pensato per futura esportazione via MQTT (es. `.../event/...`).

---

## Pinout (riassunto)

> ⚠️ Il pinout è rigido e non va modificato nel firmware.

- **Zone ADC**: GPIO1..2,4..11  
- **VBUS ADC**: GPIO14  
- **USB**: D− GPIO19, D+ GPIO20  
- **CAN (TWAI)**: TX GPIO38, RX GPIO39  
- **W5500 (SPI2)**:  
  - SCLK GPIO40, MOSI GPIO41, MISO GPIO42  
  - CS GPIO47, INT GPIO17, RST GPIO18  
- **PN532 (SPI3)**:  
  - SCLK GPIO15, MOSI GPIO16, MISO GPIO21  
  - CS GPIO48, RST TXD0, IRQ RXD0  
- **I²C (MCP23017)**: SDA GPIO12, SCL GPIO13  
- **LED_STATO**: GPIO0  

La mappatura dettagliata è in `main/pins.h`.

---

## Architettura software

- **ESP-IDF 5.x** (target `esp32s3`), C, FreeRTOS.  
- Moduli principali (file indicativi):

  - `main.c` – `app_main()`, init globali, creazione task.  
  - `cfg.c/.h` – configurazione in NVS (MQTT, claim, parametri base).  
  - `eth_w5500.c/.h` – driver W5500 + integrazione `esp-netif`.  
  - `mqtt_cli.c/.h` – bootstrap/claim + modalità device (PUB/SUB helper).  
  - `http_prov.c/.h` – web server di provisioning.  
  - `mcp23017.c/.h` – driver expander + API LED/DOUT/ingressi.  
  - `zone.c/.h` – gestione zone ADC / EOL / stati logici.  
  - `busmon.c/.h` – monitor BUS 13,8 V.  
  - `led.c/.h`, `led_panel.c/.h` – gestione LED locali/MCP.  
  - `nfc_pn532.c/.h` – NFC PN532 (init + lettura UID).  
  - `can_bus.c/.h` – bus CAN espansioni (stub + diagnostica).  
  - `event_log.c/.h` – log eventi centralizzato.

- **Task FreeRTOS** (indicativi):
  - `task_net` – porta su Ethernet/DHCP.  
  - `task_prov` – provisioning HTTP + claim MQTT.  
  - `task_mqtt` – gestione connessione device, heartbeat, comandi.  
  - `task_zone` – acquisizione ADC zone + VBUS.  
  - `task_led` – pattern LED.  
  - `task_nfc`, `task_can` – NFC e bus CAN.  

---

## MQTT & Claim (EMQX)

- Fase bootstrap:  
  - utente `nsbootstrap` (password configurata a compile-time),  
  - `client_id = device_id = nsap-<MAC>`.  
- Claim:  
  - PUB `provision/claim/<client_id>`, payload `{ device_id, claim_code }`.  
  - SUB `provision/reply/<client_id>`, attesa reply `{ status, username, password }`.  
- Fase device:  
  - utente device (`dev_user/dev_pass` da NVS),  
  - namespace `nsalarmpro/<device_id>/...`,  
  - ACL lato EMQX limitano ogni device al proprio namespace.

---

## Build e flash

Prerequisiti:

- ESP-IDF 5.x installato e configurato (`idf.py`).  
- Toolchain per ESP32-S3.  

Passi:

```bash
idf.py set-target esp32s3
idf.py menuconfig        # opzionale: impostare flash/PSRAM, logging, bootstrap password
idf.py build
idf.py -p /dev/ttyACM0 flash monitor
```

## Note

Soglie, tempi di debounce e mappatura EOL sono placeholder e vanno calibrati sul campo.
Le password reali (bootstrap, device) e il certificato CA del broker vanno impostati prima della produzione.
La gestione completa di scenari di allarme (away/home/notte/personalizzato), log su file e aggiornamenti OTA può essere aggiunta in step successivi sfruttando l’architettura modulare esistente.