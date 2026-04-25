# History flow voor Home Assistant sensoren

Dit bestand bevat een importeerbare Node-RED flow om historische data op te halen van de sensoren die je in je controller gebruikt.

## Bestand

- `ha_history_flow.json`

## Wat de flow doet

1. Trigger elke 30 minuten (en handmatig via inject).
2. Vraagt de laatste 24 uur historie op per sensor via de Home Assistant API endpoint:
   - `/api/history/period/<start>?filter_entity_id=<sensor>&end_time=<end>`
3. Normaliseert elke tijdreeks naar `{ last_changed, state }`.
4. Combineert alle 10 sensoren in één object.
5. Stuurt resultaat naar debug én schrijft snapshot naar `/data/history_snapshot.json`.

## Benodigde environment variables in Node-RED

- `HA_BASE_URL` (bv. `http://homeassistant.local:8123`)
- `HA_TOKEN` (Long-Lived Access Token uit Home Assistant)

## Sensoren in deze flow

- `sensor.p1_meter_5c2faf04849e_active_power`
- `sensor.epex_spot_data_price`
- `sensor.lilygo_rs485_3_marstek_battery_state_of_charge`
- `sensor.soc`
- `sensor.energy_production_today_remaining`
- `sensor.energy_production_tomorrow`
- `sensor.power_production_next_12hours`
- `sensor.power_production_next_24hours`
- `sensor.kotsolar_total_power`
- `sensor.p1_meter_5c2faf04849e_active_average_demand`

## Importeren

- Node-RED → menu → Import → plak inhoud van `ha_history_flow.json`.
- Deploy.
- Zet env vars en trigger de inject-node.

