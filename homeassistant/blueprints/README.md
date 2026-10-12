# Blueprint « Brink Flair - VMC intelligente »

[![Importer le blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fekozan%2Fbrink-flair-modbus%2Fblob%2Fmain%2Fhomeassistant%2Fblueprints%2Fautomation%2Fbrink_smart_ventilation.yaml)

Automatisation Home Assistant qui pilote le débit (m³/h) et le by-pass d'une Brink Flair
via l'ESPHome de ce dépôt. Elle tourne chaque minute, calcule un débit cible et n'écrit sur
le Modbus que si ce débit change d'au moins *Variation minimale* (10 m³/h par défaut).

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
| Refroidissement gratuit | T int ≥ 24 °C, T ext ≤ T int − 2 °C, T ext ≥ 12 °C | by-pass **ouvert**, by-pass boost ON (si renseigné), débit refroidissement ; s'arrête quand T int redescend à 22,5 °C |
| Protection été | T ext ≥ T int − 0,5 °C et T int ≥ 22 °C | by-pass **fermé** (l'échangeur récupère la fraîcheur), débit réduit |
| Boost humidité | dérivée ≥ 1 %/min **ou** humidité ≥ 80 % | débit boost ; maintenu au moins 10 min, puis tant que l'humidité baisse vite (< −0,3 %/min) et reste > 60 % ; 60 min maximum |
| Boost CO₂ | CO₂ > 800 ppm | débit proportionnel jusqu'au max à 1200 ppm |
| Boost pollution | capteur PM2.5 / COV > seuil bas | débit proportionnel jusqu'au max au seuil haut |
| Pollution extérieure | PM2.5 ext ≥ 35 | débit de base plafonné |
| Absence | présence OFF depuis 15 min | débit absence (boosts désactivés sauf option) |
| Vacances | interrupteur vacances ON | débit vacances |
| Nuit | plage horaire | débit plafonné, les boosts restent prioritaires |

Tous les seuils sont réglables. Le by-pass ne revient à « Auto » (ou « Fermé », au choix)
que hors refroidissement et hors protection été. Des hystérésis évitent les allers-retours
du moteur du by-pass.

## Prérequis

1. **ESPHome** : entités `number.brink_modbus_flow_rate`, `select.brink_modbus_control_mode`,
   `select.brink_bypass_mode`, `switch.brink_bypass_boost` (noms selon votre `name:`).
   Le blueprint force *Modbus Control Mode* sur **Modbus Flow** si vous renseignez ce select.
2. **Températures** : une sonde intérieure de référence (thermostat, ou la sonde de
   l'ESP sur l'extraction) et `sensor.brink_temperature_from_outside`.
3. **Helpers** (Paramètres > Appareils et services > Entrées), tous facultatifs :
   - **Dérivée** du capteur d'humidité : précision 2, fenêtre 3 min, unité de temps *minutes* ;
   - **Interrupteur** `input_boolean.vmc_boost_humidite` : mémoire du boost humidité
     (sans lui le boost s'arrête dès que l'humidité ne monte plus) ;
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
- Le comportement du *Bypass Boost* dépend du firmware : en mode Modbus Flow, c'est le débit
  envoyé par le blueprint qui prime. Laissez ce champ vide si vous préférez ne piloter que le
  *Bypass Mode*.
- Pour reprendre la main, activez l'interrupteur de pause : le blueprint ne touche plus à rien.
