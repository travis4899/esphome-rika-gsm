# ESPHome RIKA GSM — Fork ESP32-C3

Fork du projet original [n-serrette/esphome_rika_gsm](https://github.com/n-serrette/esphome_rika_gsm), qui permet d'émuler le module GSM proposé en option sur certains poêles à granulés RIKA.

Ce fork documente une installation fonctionnelle sur un **RIKA Como de première génération**, avec un **ESP32-C3 Super Mini** et un convertisseur **MAX3232 RS232 vers TTL**.

Le composant permet d'envoyer des commandes au poêle et de récupérer les messages de statut qu'il transmet. L'intégration Home Assistant ajoute une entité `climate` pour gérer la consigne et l'arrêt du poêle.

> Projet communautaire non officiel, non affilié à RIKA. Seule la configuration décrite ici a été testée.

## Schéma de câblage

Le schéma d'origine est conservé ci-dessous pour référence.

**Attention : le level shifter représenté sur le schéma est à remplacer par un MAX3232.** Le MAX3232 assure la conversion entre le signal RS232 côté poêle et les niveaux logiques TTL côté ESP32-C3.

<img src="schema/esp_rika_stove.png">

Le module GSM et la sonde de température ambiante partagent le connecteur DA15 situé à l'arrière du poêle. Le raccordement peut être réalisé avec un câble en Y ou un connecteur adapté.

**Ne pas déconnecter ni perturber la sonde de température ambiante**, qui reste utilisée par le poêle pour sa régulation.

## Matériel et configuration testés

- **Poêle :** RIKA Como, première génération
- **Carte :** ESP32-C3 Super Mini
- **Convertisseur série :** MAX3232 RS232 vers TTL
- **Débit UART :** 38 400 bauds
- **Framework :** ESPHome avec ESP-IDF
- **Supervision :** Home Assistant via l'API native ESPHome

Les autres modèles de poêles RIKA et les autres débits série ne sont pas vérifiés.

## Câblage

### Connexions d'alimentation

| Connecteur du poêle | ESP32-C3 Super Mini |
|---|---|
| DA15 broche 1 : +5 V | Entrée `5V` |
| DA15 broche 5 : GND | `GND` |

Le MAX3232 et l'ESP32-C3 doivent partager une masse commune.

Une seule broche GND est nécessaire sur l'ESP32-C3 : tous les éléments peuvent être reliés à cette même masse.

### Connexions UART

| Signal | Connexion |
|---|---|
| ESP32-C3 GPIO0 — TX | MAX3232, côté TTL, entrée `IN` |
| ESP32-C3 GPIO1 — RX | MAX3232, côté TTL, sortie `OUT` |
| DA15 broche 2 — TX du poêle dans ce montage | MAX3232, côté RS232, entrée `IN` |
| DA15 broche 3 — RX du poêle dans ce montage | MAX3232, côté RS232, sortie `OUT` |

Ces correspondances sont celles du montage testé. Vérifier le sens des signaux sur le module utilisé : les marquages et les flèches peuvent varier selon les fabricants.

### Alimentation du MAX3232

Le module utilisé dans ce montage est alimenté en **3,3 V**.

Certains modules adaptateurs ont leurs bornes positives reliées électriquement entre elles. Dans ce cas, ne jamais raccorder simultanément une alimentation 3,3 V et une alimentation 5 V à ces bornes.

Vérifier impérativement les caractéristiques du module utilisé avant le branchement.

**Ne jamais raccorder directement un signal RS232 à un GPIO de l'ESP32-C3.** Un convertisseur RS232 adapté est indispensable.

## Fonctionnement

Le composant `rika_gsm` émule le module GSM du poêle :

- Il reçoit les requêtes série du poêle.
- Il permet de lui transmettre des commandes.
- Il récupère les messages de statut que le poêle souhaite envoyer.
- Il expose ces informations dans Home Assistant via ESPHome.

Aucune communication GSM réelle n'est nécessaire. Les échanges SMS sont émulés sur la liaison série.

### Commandes utilisées

| Commande | Fonction |
|---|---|
| `ON` | Demande d'allumage |
| `OFF` | Demande d'arrêt |
| `r16` | Allumage avec une consigne de 16 °C |
| `r21` | Allumage avec une consigne de 21 °C |
| `r24` | Allumage avec une consigne de 24 °C |

Les commandes `rXX` définissent la consigne en degrés Celsius et peuvent allumer le poêle lorsqu'il est à l'arrêt.

### Messages de retour

Le poêle peut transmettre des messages tels que :

- `STOVE ON - ROOM 16 - PLEASE SAVE TEL NUMBER`
- `STOVE ON - ROOM 21 - PLEASE SAVE TEL NUMBER`
- `STOVE OFF - PLEASE SAVE TEL NUMBER`

La valeur `ROOM XX` correspond ici à la **consigne de température**, et non à la température ambiante mesurée.

Le poêle ne transmet pas de SMS spontané à chaque modification locale de la consigne. L'entité Home Assistant peut donc afficher une valeur obsolète après une modification directement sur le poêle, jusqu'à la réception d'un nouveau message.

## Configuration ESPHome

### Déclaration du composant externe

Dans ce fork, le composant est stocké localement :

```yaml
external_components:
  - source: esphome-rika-gsm/components
```

Adapter le chemin si le composant est utilisé depuis le github :

```yaml
external_components:
  - source:
        type: git
        url: https://github.com/travis4899/esphome-rika-gsm
        ref: main
    components: [ rika_gsm ]
```

### UART et composant RIKA

Le montage testé utilise GPIO0 pour TX et GPIO1 pour RX :

```yaml
uart:
  id: rika_uart_bus
  tx_pin: GPIO0
  rx_pin: GPIO1
  baud_rate: 38400
  debug:
    direction: BOTH
    dummy_receiver: false
    after:
      timeout: 100ms
    sequence:
      - lambda: UARTDebug::log_hex(
          direction, bytes, 16, esphome::StringRef());

rika_gsm:
  id: ria_gsm_component_1
  pin: "1234"
  phone_number: "+33612345678"
```

Paramètres du composant :

- `pin` : code PIN à quatre chiffres configuré pour l'option GSM du poêle.
- `id` : identifiant ESPHome du composant.
- `phone_number` : numéro utilisé lors de l'émulation des échanges SMS. Aucun SMS réel n'est envoyé par le réseau mobile.

Le numéro présenté ci-dessus est un exemple. Utiliser une valeur fictive ou un secret dans une configuration publiée.

### Capteur texte et capteur binaire

```yaml
text_sensor:
  - platform: rika_gsm
    raw_status:
      name: "Status poêle"
      id: rika_status

binary_sensor:
  - platform: rika_gsm
    gsm_status:
      name: "Status GSM"
```

- `raw_status` expose le dernier message de statut reçu du poêle.
- `gsm_status` indique si le poêle considère actuellement l'option GSM comme active.

Dans le montage testé, le poêle interroge le module environ toutes les cinq secondes. En l'absence de requête pendant une dizaine de secondes, l'état GSM peut repasser à inactif, puis redevenir actif à la prochaine requête.

### Entité climate Home Assistant

L'entité `climate` permet de piloter le poêle depuis Home Assistant :

- Le réglage de la consigne envoie `rXX`.
- Le mode arrêt envoie `OFF`.
- Le retour des messages du poêle actualise le mode et la consigne.

```yaml
climate:
  - platform: template
    id: rika_climate
    name: "Poêle RIKA"
    icon: mdi:stove

    optimistic: true
    restore_mode: RESTORE
    initial_state:
      mode: "OFF"
      target_temperature: 19 °C

    supported_modes:
      - "OFF"
      - HEAT

    visual:
      min_temperature: 15 °C
      max_temperature: 30 °C
      temperature_step: 1 °C

    set_target_temperature_action:
      - rika_gsm.send_command:
          command: !lambda |-
            return "r" + std::to_string((int) x);
      - climate.template.publish:
          id: rika_climate
          mode: HEAT
          target_temperature: !lambda "return x;"

    set_mode_action:
      - if:
          condition:
            lambda: 'return x == climate::CLIMATE_MODE_OFF;'
          then:
            - rika_gsm.send_command:
                command: "OFF"
          else:
            - rika_gsm.send_command:
                command: !lambda |-
                  return "r" + std::to_string(
                    (int) id(rika_climate).target_temperature
                  );
```

### Synchronisation de l'état à partir des messages SMS

Ajouter l'automatisation suivante à la déclaration du capteur `raw_status` pour synchroniser l'entité `climate` :

```yaml
text_sensor:
  - platform: rika_gsm
    raw_status:
      name: "Status poêle"
      id: rika_status
      on_value:
        then:
          - lambda: |-
              std::string s = x;

              // Le poêle est arrêté
              if (s.find("STOVE OFF") != std::string::npos) {
                id(rika_climate).set_mode(
                  climate::CLIMATE_MODE_OFF);
                id(rika_climate).publish_state();
                return;
              }

              // Le poêle est allumé : lire ROOM XX
              auto pos = s.find("ROOM ");
              if (s.find("STOVE ON") != std::string::npos &&
                  pos != std::string::npos) {
                int temp = atoi(s.c_str() + pos + 5);

                if (temp >= 15 && temp <= 30) {
                  id(rika_climate).set_mode(
                    climate::CLIMATE_MODE_HEAT);
                  id(rika_climate).set_target_temperature(temp);
                  id(rika_climate).publish_state();
                }
              }
```

Les messages `STOVE ON - ROOM XX` mettent le thermostat en mode chauffage et actualisent la consigne. Le message `STOVE OFF` fait passer l'entité en mode arrêt.

### Actions API ESPHome

Les actions API suivantes permettent également d'envoyer des commandes :

```yaml
api:
  encryption:
    key: !secret api_encryption_key

  actions:
    - action: turn_on
      then:
        - rika_gsm.send_command:
            command: "ON"

    - action: turn_off
      then:
        - rika_gsm.send_command:
            command: "OFF"

    - action: send_command
      variables:
        command: string
      then:
        - rika_gsm.send_command:
            command: !lambda "return command;"
```

### Envoi d'une commande depuis une automatisation

```yaml
on_...:
  then:
    - rika_gsm.send_command:
        command: "r21"
```

Le paramètre `command` est obligatoire, de type chaîne de caractères, et accepte une valeur calculée par lambda.

## Journalisation UART

Configuration utilisée :

```yaml
logger:
  level: DEBUG
  baud_rate: 0
```

La journalisation série classique est désactivée sur l'UART pour ne pas perturber les échanges avec le poêle.

Le bloc `uart.debug` affiche les données reçues et émises en hexadécimal. Il est utile pour le diagnostic et peut être conservé en fonctionnement normal.

## Dépannage

- **Aucun échange UART :** vérifier l'alimentation, la masse commune, le sens TX/RX, le câblage du MAX3232 et le débit de 38 400 bauds.
- **Communication dans un seul sens :** contrôler les connexions des deux côtés du convertisseur. Ne pas se fier uniquement aux flèches du module.
- **État GSM qui repasse à inactif :** le poêle a peut-être interrompu temporairement ses requêtes. Vérifier s'il redevient actif lors de la requête suivante.
- **Consigne non actualisée après une modification locale :** le poêle ne semble pas transmettre de message spontané pour chaque changement de consigne.
- **Erreur de compilation :** vérifier la compatibilité du composant externe avec la version d'ESPHome utilisée.

## Limites connues

- Testé uniquement sur un RIKA Como de première génération.
- Les autres modèles de poêles et les autres débits série ne sont pas vérifiés.
- La synchronisation du thermostat dépend des messages reçus du poêle.
- Le projet émule les échanges GSM/SMS, sans connexion cellulaire réelle.

## Crédits

- Projet original : [n-serrette/esphome_rika_gsm](https://github.com/n-serrette/esphome_rika_gsm)
- Le composant `rika_gsm` et le schéma de câblage initial proviennent du dépôt original.
- Ce fork documente l'adaptation au **ESP32-C3 Super Mini**, l'utilisation du **MAX3232 RS232-TTL** et l'intégration de l'entité `climate` dans Home Assistant.
