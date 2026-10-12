# Blueprint « Brink Flair - VMC intelligente »

[![Importer le blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fekozan%2Fbrink-flair-modbus%2Fblob%2Fmain%2Fhomeassistant%2Fblueprints%2Fautomation%2Fbrink_smart_ventilation.yaml)

Automatisation Home Assistant qui pilote le débit (m³/h) et le by-pass d'une Brink Flair
via l'ESPHome de ce dépôt. Elle tourne chaque minute, calcule un débit cible et n'écrit sur
le Modbus que si ce débit change d'au moins *Variation minimale* (10 m³/h par défaut).
Les hausses sont immédiates ; une baisse n'est appliquée qu'après au moins *Durée minimale
avant une baisse* (5 min par défaut) au débit actuel, pour éviter les va-et-vient.

## Logique

```
débit de base   = normal | absence | vacances
                → réduit par l'économie de calories (écart int/ext)
                → plafonné par la protection été
                → relevé par le refroidissement gratuit
                → plafonné la nuit (option)
                → plafonné si l'air extérieur est pollué (option)
demande         = max(boost humidité, boost CO₂, boost pollution)
débit cible     = max(base, demande), borné entre min et max
```

| Fonction | Déclenchement | Action |
|---|---|---|
| Économie de calories | écart T int − T ext > 12 °C | débit réduit linéairement jusqu'au « débit éco minimum » à 25 °C d'écart |
| Refroidissement gratuit | T int ≥ 24 °C, T ext ≤ T int − 2 °C, T ext ≥ 12 °C | by-pass **ouvert** + débit refroidissement en Modbus Flow ; s'arrête quand T int redescend à 22,5 °C |
| Protection été | T ext ≥ T int − 0,5 °C et T int ≥ 22 °C | by-pass **fermé** (l'échangeur récupère la fraîcheur), débit réduit |
| Boost cuisine / salle de bain (humidité) | dans **n'importe quelle pièce** suivie : dérivée ≥ 1 %/min **ou** humidité ≥ 80 % | débit boost ; maintenu au moins 10 min, puis tant qu'une pièce sèche encore vite (< −0,3 %/min) et que la plus humide reste > 60 % ; 60 min maximum |
| Boost cuisine / douche (déclencheur direct) | hotte, plaque, douche, lumière SDB… passe ON | débit boost immédiat, prolongé 15 min après l'arrêt ; un déclencheur resté ON plus de 60 min est ignoré |
| Boost CO₂ | CO₂ > 800 ppm | débit proportionnel jusqu'au max à 1200 ppm |
| Boost pollution | capteur PM2.5 / COV > seuil bas | débit proportionnel jusqu'au max au seuil haut |
| Pollution extérieure | PM2.5 ext ≥ 35 | débit de base plafonné |
| Boost manuel | `input_boolean` mis sur ON (bouton du tableau de bord) | débit boost manuel pendant 30 min, puis retour automatique (l'interrupteur repasse sur OFF) ; prioritaire sur l'absence et la nuit |
| Absence | présence OFF depuis 15 min | débit absence (boosts désactivés sauf option) |
| Vacances | interrupteur vacances ON | débit vacances |
| Nuit | plage horaire | débit plafonné, les boosts restent prioritaires |

Tous les seuils sont réglables. Le by-pass ne revient à « Auto » (ou « Fermé », au choix)
que hors refroidissement et hors protection été. Des hystérésis évitent les allers-retours
du moteur du by-pass.

## Paramètres réglables

Valeurs par défaut entre parenthèses.

**VMC Brink**
- Débit Modbus (m³/h) — `number.brink_modbus_flow_rate` *(obligatoire)*
- Mode de contrôle Modbus — `select.brink_modbus_control_mode`, maintenu sur Modbus Flow *(obligatoire)*
- Mode by-pass — `select.brink_bypass_mode` (optionnel)
- Pause / mode manuel — `input_boolean` (optionnel)
- Boost manuel — `input_boolean` (optionnel), durée (30 min), débit (280 m³/h)
- Texte d'état — `input_text` (optionnel)

**Débits**
- Débit minimum absolu (50 m³/h) / maximum absolu (280 m³/h)
- Débit normal en présence (150 m³/h)
- Débit absence (90 m³/h) / vacances (60 m³/h)
- Variation minimale avant écriture Modbus (10 m³/h)
- Durée minimale avant une baisse de débit (5 min) : les hausses sont immédiates, les baisses attendent que le débit actuel soit tenu au moins ce temps

**Températures, économie de calories et été**
- Température intérieure, température extérieure *(obligatoires)*
- Écart int/ext de début de réduction (12 °C) et de réduction maximale (25 °C)
- Débit éco minimum par grand froid (80 m³/h)
- Protection été : activée (oui), T int minimale (22 °C), marge (0,5 °C), débit (70 m³/h)

**Refroidissement gratuit (by-pass)**
- Activé (oui)
- T int de démarrage (24 °C) / d'arrêt (22,5 °C)
- Écart minimal int − ext (2 °C), T ext minimale (12 °C)
- Débit refroidissement (250 m³/h)
- Refroidir aussi en absence (oui), plafond de nuit appliqué au refroidissement (non)
- Position du by-pass le reste du temps : Auto ou Fermé (Auto)

**Boost cuisine / salle de bain**
- Capteurs d'humidité et dérivées en %/min, **plusieurs possibles** (SDB, cuisine…) ; la pièce la plus humide compte
- Déclencheurs directs (hotte, plaque, douche, lumière SDB…), plusieurs possibles
- Temporisation après arrêt d'un déclencheur direct (15 min)
- Mémoire du boost `input_boolean` (optionnel, recommandé)
- Seuil de montée (1 %/min), seuil de stabilisation (0,3 %/min)
- Humidité absolue de boost (80 %), humidité de fin (60 %)
- Durée minimale (10 min) / maximale (60 min)
- Débit boost cuisine / salle de bain (280 m³/h)

**Boost CO₂ et pollution**
- Capteur CO₂ (optionnel), début de boost (800 ppm), boost maximal (1200 ppm)
- Capteur de pollution intérieure (optionnel), seuil bas (15) / haut (50)
- Débit boost CO₂ / pollution maximal (280 m³/h)
- Capteur de pollution extérieure (optionnel), seuil (35), débit max si pollué (70 m³/h)

**Absence, vacances et nuit**
- Entité de présence (optionnel), délai avant mode absence (15 min)
- Entité vacances (optionnel)
- Autoriser les boosts humidité / CO₂ en absence (non)
- Plafond de nuit : activé (non), début (22:30), fin (07:00), débit max (120 m³/h)

## Prérequis

1. **ESPHome** : entités `number.brink_modbus_flow_rate`, `select.brink_modbus_control_mode`
   et `select.brink_bypass_mode` (noms selon votre `name:`).
   Le blueprint maintient *Modbus Control Mode* sur **Modbus Flow** : tout le pilotage passe
   par le débit en m³/h. La fonction *Bypass Boost* de la Brink n'est pas utilisée
   (laissez-la désactivée).
2. **Températures** : une sonde intérieure de référence (thermostat, ou la sonde de
   l'ESP sur l'extraction) et `sensor.brink_temperature_from_outside`.
3. **Helpers** (Paramètres > Appareils et services > Entrées), tous facultatifs :
   - une **Dérivée** par capteur d'humidité (SDB, cuisine) : précision 2, fenêtre 3 min,
     unité de temps *minutes* ;
   - pour une hotte sur prise connectée : un **Capteur de seuil** sur la puissance
     (ex. > 30 W) à utiliser comme déclencheur direct ;
   - **Interrupteur** `input_boolean.vmc_boost_humidite` : mémoire du boost humidité
     (sans lui le boost s'arrête dès que l'humidité ne monte plus) ;
   - **Interrupteur** `input_boolean.vmc_boost_manuel` : boost manuel temporisé (bouton sur le tableau de bord) ;
   - **Interrupteur** `input_boolean.vmc_pause` : suspend l'automatisation (mode manuel) ;
   - **Interrupteur** `input_boolean.vacances` ;
   - **Texte** `input_text.vmc_etat` (max 255) : affiche le mode actif, ex.
     `présence, éco Δ15.0°C, boost humidité → 280 m³/h`.
4. **Présence** : `zone.home` (nombre de personnes), un `group`/`person`, ou un
   `binary_sensor`/`input_boolean`.

## Installation

Cliquez sur le badge ci-dessus, ou copiez
`automation/brink_smart_ventilation.yaml` dans `config/blueprints/automation/brink/`,
puis *Paramètres > Automatisations > Blueprints > Brink Flair - VMC intelligente*.

## Remarques

- Réglez « Débit maximum absolu » selon votre modèle (le slider ESPHome de la Flair 325
  est limité à 280 m³/h).
- La protection antigel reste gérée par la Brink elle-même.
- Pour reprendre la main, activez l'interrupteur de pause : le blueprint ne touche plus à rien.
