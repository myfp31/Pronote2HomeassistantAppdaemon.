# Pronote2HomeassistantAppdaemon.
How to integrate Pronote2Homeassistant with HA OS &amp; AppDaemon

## contexte

Le but de ce tutoriel est d'intégrer Pronote2Homeassistant (https://github.com/dathosim/Pronote2Homeassistant) dans HomeAssistant OS.
Etant donné que l'intégration de python standard dans HA ne permet pas les "import" ( voir la note sur cette page https://www.home-assistant.io/integrations/python_script/ ) et n'a pas nativement de crontab accéssible, j'ai préféré intégrer ce projet via AppDaemon.

## Prerequis

J'ai suivi l'intégration proposé par rem81 pour AppDaemon : https://domo.rem81.com/2022/02/05/ha-addon-appdaemon-programme-python/

Je considère par la suite que cette intégration est fonctionnelle.

## Integration

AppDaemon fonctionne sur des classes et non directement sur un script.
La classe doit comporter une méthode initialize qui sera exécutée une fois au démarrage et qui lance les autres classes.
J'ai choisi d'ordonnancer tout depuis mon script python. Il aurait été possible aussi de le faire à partir d'un état dans HomeAssistant.
Mon ordonnancement est le suivant :
  - Dans la méthode initialize, lancement toutes les heures à 37 ( 7h37, 8h37...) la méthode check_day. pour changer le moment modifiez la ligne runtime
  - Dans la méthode check_day, lancement entre 7H et 20H les jours travaillés et uniquement à 7h et 20h, les jours non travaillés.
  - le script principal ira donc dans la méthode request

J'utilise la librairie workcalendar pour les jours fériés en France.
> passage vers vacances-scolaires-france serait certainement plus logique.

### Configuration de AppDaemon

nous aurons besoin de deux librairies complémentaires, nous modifions donc dans HA, la configuration AppDaemon :
``` yaml
python_packages:
  - pronotepy
  - workalendar
```

### Mise en place de l'application

  - Dans le fichier appdaemon.yaml sous \config\appdaemon, ajoutez la définition du log pour l'application :
``` yaml
  pronote_log:
    name: PronoteLog
    filename: /config/log/pronote.log
    log_generations: 3
    log_size: 200000
```

  - Créez le fichier pronote.yaml dans \config\appdaemon\apps
  
 ``` yaml
---
pronote:
  module: pronote
  class: get_data
```

  - Enfin créez le fichier pronote.py dans \config\appdaemon\apps ainsi :
``` python 
import appdaemon.plugins.hass.hassapi as hass
from ast import If
import pronotepy
from pronotepy.ent import ac_lyon
from pronotepy.ent import ac_grenoble
from pronotepy.ent import ac_orleans_tours
from pronotepy.ent import ac_reims
from pronotepy.ent import ac_reunion
from pronotepy.ent import atrium_sud
from pronotepy.ent import ile_de_france
from pronotepy.ent import monbureaunumerique
from pronotepy.ent import occitanie_montpellier
from pronotepy.ent import paris_classe_numerique


import os
import sys
from datetime import datetime, timedelta, time, date
from workalendar.europe import France
import json
# ci-dessus import identiques à pronote2HomeAssistant avec en plus hass, datetime et workcalendar

# definition de la class
class get_data(hass.Hass):
  # definition du log à utiliser
	classlog = "pronote_log"
	
	def initialize(self):
		self.log("Hello from AppDaemon pronote app", log=self.classlog)
		self.log("Hello from AppDaemon pronote app : %s", pronotepy.__version__,  log=self.classlog)
		runtime = time(0, 37, 0)
		self.run_hourly(self.check_day, runtime )
	
	def check_day(self, kwargs):
		Thedatetime = datetime.now()
		cal = France()
		if cal.is_working_day(Thedatetime) :
			if int(Thedatetime.hour) > 6 and int(Thedatetime.hour) < 20 :
				self.run_in(self.request, 5, Thedatetime = Thedatetime )
		elif int(Thedatetime.hour) == 7 or int(Thedatetime.hour) == 19 : 
			self.run_in(self.request, 5, Thedatetime = Thedatetime )
	
	def request(self, kwargs):
		self.log("Starting to check pronote!", log=self.classlog)
    
    #Copier la suite du script pronote.py à partir de la définition des variables. Attention à l'indentation.
    #Variables a remplacer (ou laisser comme ça pour tester la démo).....
    
    
    
    
 ```
 
ne pas oublier de relancer AppDaemon

### Autres adaptations possibles ou nécessaires  
 
 #### Configuration.yaml
 
 Dans ce chapitre, https://github.com/dathosim/Pronote2Homeassistant#2-configuration-yaml-pour-r%C3%A9cup%C3%A9rer-lemploi-du-temps-dans-un-sensor , il est nécessaire de modifier l'adresse IP pour lire le fichier json. L'adaptation écrit le fichier dans le répertoire www de appDaemon et non de HA.... il est donc nécessaire d'apter le configuration.yaml avec le port de AppDaemon soit 192.168.XX.XX:5050 en remplaçant les XX.XX avec votre bonne IP.
 
 #### print à remplacer
 
 Les prints n'apparaissent plus, il est donc plus facile d'utiliser les logs pour savoir si cela fonctionne bien 
 une possibilité est donc de définir des logs ainsi pour le debug 
 ``` python
 self.log("votre message", log=self.classlog)
 ``` 
 
 #### ent non mis à jour
 
Visiblement, je n'avais pas, dans la version publiée de pronotepy, les dernières mises à jour des différents sites ent même si j'avais une version 2.4.0.
Temporairement, j'ai donc copié l'ensemble de pronotepy depuis github, copier le répertoire pronotepy sous \config\appdaemon\apps\pronotepy et changer ma configuration appDaemon pour charger les modules nécessaire à pronotepy : 
``` python
python_packages:
  - beautifulsoup4
  - pycryptodome
  - requests
  - workalendar
```
ne pas oublier de relancer AppDaemon
