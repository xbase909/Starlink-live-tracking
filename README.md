# Starlink Sat Finder

**Quel satellite Starlink utilise mon antenne, en ce moment ?**
*Identify, in real time, which Starlink satellite your dish is connected to, and watch it on a live 3D globe.*

L'antenne Starlink ne dit jamais à quel satellite elle est connectée. Ce projet le retrouve toutes les 15 secondes, en comparant ce que l'antenne « voit » avec les orbites publiques de toute la constellation, et l'affiche sur un globe 3D en direct.

## Ce que ça fait

- **`starlink-live`** : identifie le satellite en cours d'utilisation à chaque changement (toutes les 15 s) et publie le résultat dans `data/live.json`.
- **`starlink-globe`** : page web avec un globe 3D :
  - toute la constellation (~10 000 satellites) qui se déplace en temps réel ;
  - en bleu, les satellites visibles depuis ton antenne ;
  - le satellite en cours, relié à l'antenne, avec un badge **EN DIRECT** ;
  - les derniers satellites utilisés ;
  - trois fonds au choix : imagerie satellite, nuit, plan.
- **`sat_finder.py`** en ligne de commande : une capture ponctuelle (1 minute par défaut) avec un tableau des satellites identifiés.
- **`sat_map.py`** : une carte 2D (Leaflet) des derniers satellites identifiés et de leur trace au sol.

## Comment ça marche

1. Starlink change de satellite à des instants fixes : aux secondes 12, 27, 42 et 57 de chaque minute.
2. À chaque changement, le programme **efface la carte d'obstruction** de l'antenne, puis la relit chaque seconde pendant 14 s. Les nouveaux pixels qui s'allument dessinent la trajectoire du satellite utilisé.
3. Cette trajectoire est convertie en directions dans le ciel (élévation, azimut), puis comparée à la position calculée de tous les satellites Starlink, à partir des données orbitales publiques de [CelesTrak](https://celestrak.org). Le plus proche est retenu.

Le **score** affiché mesure l'écart entre le trait observé et l'orbite retenue : en dessous de 5, la correspondance est excellente ; en dessous de 10, elle est bonne. La **marge** est l'écart avec le deuxième candidat : plus elle est grande, moins il y a d'ambiguïté.

Méthode issue des travaux de l'Université de Victoria : *Trajectory-based Serving Satellite Identification with User Terminal's Field-of-View*, LEO-NET'24 ([article](https://onlineacademiccommunity.uvic.ca/starlink/wp-content/uploads/sites/8876/2024/09/leonet24-ali.pdf), [code d'origine SatInView](https://github.com/aliahan/SatInView)). Ce projet en est une réimplémentation simplifiée, fonctionnant en continu et en conteneur.

## Prérequis

- Une antenne Starlink **fixe**, avec une carte d'obstruction en repère `FRAME_EARTH` (vérifié au démarrage).
- Un hôte Docker capable de joindre l'antenne sur `192.168.100.1:9200`. Si l'application Starlink fonctionne depuis ton réseau, c'est bon. Avec ton propre routeur, il faut généralement une route vers `192.168.100.0/24`.
- La position de l'antenne (latitude, longitude). Depuis mai 2026, l'antenne ne la fournit plus elle-même.

## Installation

```bash
git clone https://github.com/<ton-compte>/starlink-sat-finder.git
cd starlink-sat-finder
cp .env.example .env
nano .env                # renseigne DISH_LAT et DISH_LON
docker compose up -d --build
docker logs -f starlink-live
```

Une ligne s'affiche toutes les 15 secondes :

```
13:19:57  STARLINK-32465       469 km  score 2.8
```

La page est ensuite disponible sur `http://<ip-hote>:8090`. Pour la publier derrière Traefik, les labels sont prêts en commentaire dans `docker-compose.yml`.

### Capture ponctuelle (sans le mode live)

```bash
docker compose run --rm starlink-live sat_finder.py -n 8      # 8 créneaux = 2 minutes
docker compose run --rm starlink-live sat_map.py -l 8         # carte 2D -> data/carte-satellites.html
```

## ⚠️ À savoir

- **La carte d'obstruction est effacée toutes les 15 secondes** tant que le mode live tourne. L'application Starlink ne peut donc plus afficher d'obstacles pendant ce temps. Rien n'indique que la connexion en souffre, mais surveille ta latence et tes pertes : en cas de doute, `docker stop starlink-live`.
- L'antenne est interrogée chaque seconde pendant les captures. C'est du trafic local léger, environ 60 Ko/s entre l'hôte et l'antenne, sans aucun trafic Internet.
- Les orbites sont téléchargées depuis CelesTrak au plus une fois toutes les 6 heures, conformément à leur demande.
- Projet non officiel, sans lien avec SpaceX ou Starlink.

## Crédits

- [starlink-grpc-tools](https://github.com/sparky8512/starlink-grpc-tools) (Unlicense) : client gRPC de l'antenne, image Docker de base.
- [SatInView](https://github.com/aliahan/SatInView) et l'article LEO-NET'24 : méthode d'identification et étalonnage de la carte d'obstruction.
- [Skyfield](https://rhodesmill.org/skyfield/) et [satellite.js](https://github.com/shashwatak/satellite-js) : calcul des orbites.
- [globe.gl](https://globe.gl) : globe 3D. Fonds : Esri World Imagery, CARTO / OpenStreetMap, NASA Black Marble.
- Données orbitales : [CelesTrak](https://celestrak.org).

## Licence

MIT, voir [LICENSE](LICENSE).
