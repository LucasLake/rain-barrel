# Regenton

Een ESPHome + Home Assistant-project dat een regenton automatisch laat
bijspringen als watervoorziening voor de tuin/sproeiers. Zodra er
downstream water wordt afgenomen (gedetecteerd via een flow-sensor),
schakelt dit systeem een pomp + magneetventiel in om water uit de
regenton te leveren — mits de ton niet leeg is. Met ingebouwde
veiligheden tegen lekkage, een vastgelopen pomp en ooververhitting.

> **Status:** dit is een extractie van een privé Home Assistant-setup,
> bedoeld om anderen dit na te laten bouwen. Zie [Nog te doen](#nog-te-doen)
> voor wat er nog ontbreekt (vooral: foto's/schema van de fysieke opstelling).

## Hoe het werkt

1. Een **flow-sensor** in de hoofdwaterleiding naar de tuin/sproeiers
   detecteert wanneer er water wordt afgenomen.
2. Zodra er flow is (> 1 L/min) én de **vlotter** (float switch) in de
   regenton aangeeft dat er water is, gaat het **magneetventiel** open
   en (2 seconden later) de **pomp** aan — die levert dan regenwater
   in plaats van (of naast) leidingwater.
3. Zodra de flow stopt (< 1 L/min), gaan pomp en ventiel weer uit.
4. Loopt de ton tijdens gebruik leeg (vlotter → uit), dan stoppen pomp
   en ventiel direct, en krijg je een melding. Komt er weer water bij
   (vlotter → aan), dan ook een melding.
5. Een aantal losse veiligheidsautomations bewaakt de rest:
   lekkagedetectie (te hoge flow), een pomp die te lang aanstaat, het
   magneetventiel dat niet synchroon loopt met de pomp, en een te
   warme ESP32-behuizing.

## Benodigde hardware (uit `esphome/regenton.yaml`)

| Component | Rol | GPIO | Aansluiting (HA-domein) |
|---|---|---|---|
| ESP32 dev board (`esp32dev`) | brein van het systeem | — | — |
| Pulse-based waterflow-sensor, gekalibreerd op ~444 pulsen/liter | meet doorstroom in L/min | GPIO13 | `sensor` |
| Vlotter / float switch (schakelt tegen GND) | detecteert of de ton water bevat | GPIO5 | `binary_sensor` |
| Relais (actief-laag) + **Normally Open** magneetventiel | laat regenwater door (dicht = stroom erop) | GPIO19 | `switch` |
| Relais (actief-laag) + waterpomp | pompt water uit de ton | GPIO21 | `switch` |
| PWM-ventilator (via LEDC-output) | koelt de behuizing, lokaal aangestuurd op >65°C | GPIO23 | `fan` |
| — (ESP32-interne temperatuursensor) | bewaakt behuizingstemperatuur | intern | `sensor` |

De ventilatoraansturing gebeurt volledig lokaal op de ESP (elke 2
minuten temperatuur checken, aan boven 65°C) — dat blijft dus ook
werken als Home Assistant offline is. HA krijgt via de "Regenton te
warm"-automation alleen een melding als dat gebeurt.

## Installatie

1. **ESPHome-firmware flashen**: kopieer
   [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) naar
   `esphome/secrets.yaml` en vul je eigen wifi-gegevens + een zelfgekozen
   OTA-wachtwoord in. Flash daarna
   [`esphome/regenton.yaml`](esphome/regenton.yaml) naar je ESP32 (via
   `esphome run regenton.yaml`, of upload de map via de ESPHome-add-on
   in Home Assistant).
2. Voeg het device toe aan Home Assistant via de ESPHome-integratie
   (auto-discovery via zeroconf, of handmatig met IP-adres).
3. Noteer hoe de entities heten na toevoegen (bv. `sensor.regenton_waterflow`,
   `binary_sensor.regenton_vlotter`, `switch.regenton_pomp`,
   `switch.regenton_magneetventiel`, `fan.regenton_ventilator`,
   `sensor.regenton_esp32_temperatuur`) — pas de entity-ID's in
   `automations.yaml` en `template_sensors.yaml` hierin aan als jouw
   namen afwijken.
4. Kopieer de inhoud van [`automations.yaml`](automations.yaml) naar
   je eigen `automations.yaml` (of importeer losse automations via de
   UI).
5. Kopieer de `template:`-sectie uit
   [`template_sensors.yaml`](template_sensors.yaml) naar je
   `configuration.yaml` (samenvoegen met een eventuele bestaande
   `template:`-sleutel, YAML staat geen dubbele top-level keys toe).
6. **Vervang de placeholder-notificatieservice** — zoek in
   `automations.yaml` naar `notify.YOUR_NOTIFY_TARGET` en vervang dat
   door jouw eigen notify-service (bv. `notify.mobile_app_telefoon`).
   Zie Instellingen → Automatiseringen → nieuwe actie → "Notificatie
   verzenden" in de UI om te zien welke service jij hebt.
7. Herlaad automations (Instellingen → Systeem → YAML herladen →
   Automatiseringen) en herstart HA voor de nieuwe template-sensoren.
8. Optioneel: voeg de kaart uit [`dashboard.yaml`](dashboard.yaml) toe
   aan een dashboard (Lovelace: kaart toevoegen → YAML-modus → plak de
   inhoud).

## Veiligheidsautomations

- **Regenton lekkage noodstop** — flow boven 6 L/min duidt op een lek;
  pomp direct uit, ventiel dicht, melding. Pas de drempel (`above: 6`)
  aan op je eigen normale doorstroomsnelheid.
- **Regenton pomp maximaal 21 minuten aan** — voorkomt een vastgelopen
  pomp die eindeloos doorloopt. Pas de duur aan naar wat voor jouw tuin
  realistisch is.
- **Regenton pomp en ventiel synchroon houden** — vangnet: als de pomp
  aan staat moet het ventiel ook aan (open) staan.
- **Regenton te warm** — melding als de ESP32-behuizing boven 65°C
  komt.

## Nog te doen

- [ ] Bevestigen/aanvullen van exacte inkoop-onderdelen (merk/type
      flow-sensor, pomp, magneetventiel, relais)
- [ ] Foto's/schema van de fysieke opstelling en bedrading
- [ ] Licentie kiezen (deze repo heeft er nog geen)
- [ ] Naar de uiteindelijke GitHub-repo verhuizen zodra die is aangemaakt
