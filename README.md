MetroSim

# README - Projet Simulateur de Quai de Métro en GO - MetroSim - BACKEND

## Description du Projet

Bienvenue dans le simulateur de quai de métro en GO ! Ce projet vise à reproduire le fonctionnement basique d'un quai de métro, avec des trains qui arrivent et partent à intervalles réguliers. La simulation a pour objectif d'identifier les points sensibles (ex : goulots d'étranglement) au sein d'un quai de métro.

## FRONTEND

Disponible dans le dossier /front-end. 

Pour son utilisation, il faut d'abord lancer le back-end comme expliqué ci-dessous, puis suivre la procédure du front-end (disponible dans le README du dossier).

Le FRONTEND et le BACKEND doivent donc, bien évidemment, être lancés en parallèle.

## Fonctionnalités

Le programme prend en charge plusieurs paramètres pour personnaliser son comportement. Vous pouvez spécifier ces paramètres lors de l'exécution en utilisant l'option correspondante. Voici la liste des paramètres disponibles :


- **flow**: Débit de création d'agent toutes les flow millisecondes (par exemple, "1000").
- **controleurs**: Activation des contrôleurs (par exemple, "false").
- **fraudeurs**: Activation des fraudeurs (par exemple, "false").
- **impolis**: Activation des comportements impolis (par exemple, "false").
- **mob_reduite**: Activation de la mobilité réduite (par exemple, "false").
- **patients**: Activation des comportements patients (par exemple, "false").
- **leftTopCorners**: Coins supérieurs gauche des voies de métro (par exemple, "[[9,0],[11,0]]").
- **rightDownCorners**: Coins inférieurs droit des voies de métro (par exemple, "[[10,39],[12,39]]").
- **goToLeft**: Direction vers la gauche  (sens des métros) (par exemple, "[true,false]").
- **gates**: Portes d'entrée (par exemple , "[[[8, 5], [8, 6], [8, 34]],[[13, 5], [13, 6], [13, 34]]]", respectivement les portes du métro 1 et du métro 2).
- **capacity**: Capacité des métros (par exemple, "[10,10]").
- (**maxDuration**: Durée maximale en secondes (par exemple, "600") :  :warning: non fonctionnel, mais demandée pour la configuration)
- (**frequency**: Fréquence (par exemple, "[10,10]") : :warning: non fonctionnel, mais demandée pour la configuration)
- (**stopTime**: Temps d'arrêt (par exemple, "[5,5]") : :warning: non fonctionnel, mais demandée pour la configuration)


## Lancement du BACKEND

```bash 
go run "[chemin_absolu]\main.go"
```

## Interface directe avec l'API

Après execution de main.go :

### Configuration de la simulation

```bash 
http://127.0.0.1:12000/configure
```

(voir fichier cfg_request.go pour le format attendu d'une configuration)


![Exemple de requête HTTP POST correcte](https://gitlab.utc.fr/pillisju/metrosim/-/blob/main/request_image.png)

### Lancement de la simulation, et récupération de son état

Une fois la configuration envoyée, il suffit de faire une requête GET sur l'URL suivante :

```bash 
http://127.0.0.1:12000/launch
```

Pour le rafraichissement de la simulation, une nouvelle requête GET vers cette URL suffit.

## Bugs Connus

Le projet fonctionne globalement. La configuration de la fréquence des trains et de leur temps d'arrêt n'est pas actuellement prise en charge (possible mais non fonctionnel, comportement anormal des metros). Ces paramètres sont fixés à des valeurs prédéfinies pour le moment (5 secondes).

## Contribution 

Projet académique réalisé dans le cadre de l'UV AI30 à l'Université de Technologie de Compiègne. **Décembre 2023**

4 membres : 
- **Yohan Folliot**
- **Jana Rafei**
- **Julien Pillis**
- **Yousra Hassan**