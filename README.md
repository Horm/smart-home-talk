# Smart Home Talk
Zdroje pro "live coding demo" v rámci přednášky ["Smart Home Trochu jinak"](https://docs.google.com/presentation/d/1UTx008-0oevkyYd32_VD9X9rcYITUUmxR5z1AW7NzI0/edit?usp=sharing)...

## Home Assistant

Pokud instalujete na Raspberry Pi, doporučuji jako hassbian image, ale je možný i Docker: 

```
docker run -d --name="home-assistant" -v `pwd`/config:/config -e "TZ=Europe/Prague" -p 8123:8123 homeassistant/home-assistant
```

Tímto příkazem si vytvoříme docker container s prolinkovanou složkou konfigurace. Pokud vše proběhlo správně, měla by se vám vytvořit složka `config` a na `localhost:8123` by platforma měla běžet. 

## Sensory

### Světla

Pouze jako demo. Do `configuration.yaml` vložíme tyto řádky:

```yaml
light:
  - platform: demo
```

Zkontrolujeme konfiguraci a restartujeme platformu...

### Teplota, Termostat, Spínání kotle...

Pod `sensor:` přidáme:
```yaml
  - platform: demo
  - platform: demo
```
Zkontrolujeme konfiguraci a restartujeme platformu...

Vytvoříme průměrnou teplotu. Opět pod `sensor:` přidáme:

```yaml
  - platform: template
    sensors:
      average_temperature:
      unit_of_measurement: "°C"
      value_template: >-
        {{ (float(states.sensor.outside_temperature.state) + float(states.sensor.outside_temperature_2.state)) / 2 | round(2) }}
```
Zkontrolujeme konfiguraci a restartujeme platformu...

Zobrazila se vám chyba v konfiguraci? Pokud ano, opravíme a opět restartujeme...

Přidáme komponentu pro spínání kotle (normálně jako GPIO, nyní jako boolean komponentu):
```yaml
input_boolean:
  heating_enabled:
```

Zkontrolujeme konfiguraci a restartujeme platformu...

Přidáme termostat: 
```yaml
climate:
  - platform: generic_thermostat
    name: Thermostat
    heater: input_boolean.heating_enabled
    target_sensor: sensor.average_temperature
    min_temp: 10
    target_temp: 26
    cold_tolerance: 0.5
    hot_tolerance: 0
    initial_operation_mode: "auto"
```
Jméno termostatu tvoří pak následně jeho `entity_id`. 

Zkontrolujeme konfiguraci a restartujeme platformu...

## Skupiny

Některé skupiny jsou v základu vytvořené nebo si můžeme vytvářet vlastní v souboru `groups.yaml`: 
```yaml
thermostats:
  view: false
  entities: 
    - climate.thermostat
```
## Uživatelské rozhraní

Pro GUI využijeme modul `Lovelace GUI`. Ten nám dovolí GUI konfigurovat opět pomocí psané konfigurace...

Přidáme:
```yaml
lovelace:
  mode: yaml
  ```

A vytvoříme nový soubor `ui-lovelace.yml`:
```yaml
title: Our Home
views:
  - title: Home
    path: default
    cards: 
      - type: thermostat
        entity: climate.thermostat
        name: Bedroom
      - type: horizontal-stack
        cards:
          - type: light
            name: Bedroom
            entity: light.bed_light
          - type: light
            name: Kitchen
            entity: light.kitchen_lights
      - type: history-graph
        entities: 
          - entity: sensor.average_temperature
            name: Average Temperature
          - entity: sensor.outside_temperature
            name: Outside Temperature
        hours_to_show: 48
        title: Temperature last 2 days
  - title: Domov
    path: default
    cards: 
      - type: thermostat
        entity: climate.thermostat
        name: Ložnice
      - type: horizontal-stack
        cards:
          - type: light
            name: Ložnice
            entity: light.bed_light
          - type: light
            name: Kuchyně
            entity: light.kitchen_lights
      - type: history-graph
        entities: 
          - entity: sensor.average_temperature
            name: Průměrná teplota
          - entity: sensor.outside_temperature
            name: Venkovní teplota
        hours_to_show: 48
        title: Teploty za poslední 2 dny
        
```

Zkontrolujeme konfiguraci a restartujeme platformu (a možná budeme muset v nastavení Lovelace aktivovat)...

## Automatizace

### Spuštění na základě změny stavu
```yaml
- alias: 'Light state change sets wanted temperature'
  trigger:
    platform: state
    entity_id: light.bed_light
  action: 
    - service: climate.set_temperature
      data_template:
        entity_id: 
          - group.thermostats
        temperature: 22
```

### Spuštění na základě změny času...
```yaml
- alias: 'Afternoon'
  trigger:
    platform: time_pattern
    hours: '/1'
  condition:
    condition: 'time'
    after: '16:00:00'
    before: '22:00:00'
  action: 
    - service: climate.set_temperature
      data_template:
        entity_id: 
          - group.thermostats
        temperature: 25
```

## Export Dat

```bash
$ sqlite3 config/home-assistant_v2.db
```

```sql
select state,created from states where entity_id = "sensor.average_temperature";
```
