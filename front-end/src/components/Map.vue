<script>
  import * as PIXI from 'pixi.js' ;
  import * as couleurs from "@/assets/couleurs.json"
  import BoxOptions from "@/components/Box-options";
  import {max, min} from "@popperjs/core/lib/utils/math";

  // Variables ne nécessitant pas d'être traitées comme partie du composant.
  // ~~> Variables statics

  // Dictionnaire contenant les types de cases correspondants aux couleurs
  // Sera utilisé pour savoir quel type de case à été cliqué.
  // L'élément PixiJS cliqué lui ne peut contenir que l'information sur sa couleur.
  // Graphiquement c'est donc la couleur qui contient l'information
  let couleursInverse = {}
  for (let couleursKey in couleurs) {
    couleursInverse[couleurs[couleursKey]] = couleursKey
  }

  /* eslint-disable */
  const width = 700;
  const height = 700;
  const nbRect = 50;
  const longRect = width / nbRect;
  const hautRect = height / nbRect;

  // Une case à t'elle déjà été cliqué ?
  let selectCase = false

  // Première case cliquée
  let firstCase = null
  // Deuxième case cliquée
  let secondCase = null

  // Dictionnaire contenant le lien vers le Sprite PixiJS en fonction de ses coordonnées
  let matriceMapRectanglesFromCoord = [[]]

  // Permet de savoir si une fonction doit être appelé ou non.
  // Les Eventlistener ne doivent être associés que lors de l'initialisation des Sprite.
  // Et non pas réassociés lors d'une modification par exemple
  let initialized = false

  export default {
    name: "Map-simulation",
    components: {BoxOptions},
    data() {
      return {
        map: [],                    // Map contenant les types de cases // Sera comparée avec celle du back-end
        mapOriginale: [],           // Map de départ avant modification // Unused
        color: '0xE9E9E9',          // Couleur actuellement sélectionnée lors de la création d'une map
        creationMap: [],            // Map temporaire utilisée lors de la création de la map
        modification: true,         // Etat du front-end. Mode création de map ou Mode affichage de simulation en cours
        simulationAlive: false,     // La simulation est-t'elle active
        couleurs: couleurs,         // Dictionnaire des couleurs en fonction de leur type de case
        couleursInverse: couleursInverse,       // Dictionnaire des cases en fonction de leur couleur
        textEtapeCreation: ["Création des murs et couloirs", "Plaçage des voies", "Plaçage des portes"],
        etapeCreation: 0,           // Etape lors de la création
        selectedVoie: 1,            // Voie sélectionnée que l'on va placer
        selectedPortesVoie: 1,      // Voie sélectionnée sur laquelle on ajoutera les portes
        voie1: Array(4),  // Point haut-gauche et bas droit de la première voie
        voie2: Array(4),  // Point haut-gauche et bas droit de la seconde voie
        texteErreur: "",
        portesMetro: [[],[]],       // Coordonnées de chaque porte de voie [[portes voie 1,...], [porte voie 2,...]]
        paramTab: 1,                // Onglet des paramétrages actif
        frequence: 15,              // Fréquence des trains
        capacite: 50,               // Capacité de trains
        stop: 15,                   // Temps d'arrêt des trains
        params: {                   // Objet contenant les paramètres, bindé avec les inputs
          map: [],                  // Map à envoyer ay back-end
          maxDuration: 150,         // Durée maximale de la simulation
          flow: 1,                  // Nb d'agents créés par milliseconde
          fraudeurs: true,          // Agents fraudeurs activés/désactivés
          controleurs: true,        // Agents controleurs activés/désactivés
          impolis: true,            // Agents impolis activés/désactivés
          mob_reduite: true,        // Agents à mobilité réduite activés/désactivés
          patients: true,           // Agents patients activés/désactivés
          frequency: [],            // Fréquence des trains pour les deux voies
          stopTime: [],             // Temps d'arrêt des trains pour les deux voies
          capacity: [],             // Capacité de trains pour les deux voies
          leftTopCorners: [],       // Coin haut-gauche de chaque voie
          rightDownCorners: [],     // Coin bas-droite de chaque voie
          goToLeft: [],             // Sens des trains
          gates: []                 // Coordonnées de chaque porte de voie [[portes voie 1,...], [porte voie 2,...]]
        }
      }
    },
    methods: {
      /***
       * Fonction peuplant le CANVAS PixiJS avec le contenu de la map
       *
       * Elle se contente de réafficher le canvas
       * d'affichage (en opposition avec celui de création)
       */
      drawPixi() {
        const canvas = document.getElementById('pixi');
        const app = new PIXI.Application({
          width: width,
          height: height,
          antialias: true,
          transparent: true,
          view: canvas,
        })

        app.sortableChildren = true

        // Partie pour l'affichage des Sprites
        //let matriceMapRectangles = []
        //const txt = PIXI.Texture.from(require("@/assets/Basic_red_dot.png"))
        //const wall = PIXI.Texture.from(require("@/assets/map/wall.png"))
        //const wallTexture = PIXI.Sprite.from("wall.png");

        for (let i = 0; i < this.map[0].length; i++) {
          for (let j = 0; j < this.map.length; j++) {

            let rectangle = new PIXI.Sprite(PIXI.Texture.WHITE);

            rectangle.position.set(i * longRect, j * hautRect)
            rectangle.width = longRect;
            rectangle.height = hautRect;
            rectangle.tint = couleurs[this.map[j][i]]

            // let round = new PIXI.Sprite(txt)
            //
            // round.position.set(i * longRect, j * hautRect)
            // round.width = longRect;
            // round.height = hautRect;
            //
            // //round.position.set(i * longRect, j * hautRect)
            // // round.zIndex = 1
            // // rectangle.sortableChildren = true
            // app.stage.addChild(round)

            matriceMapRectanglesFromCoord[`${i},${j}`] = rectangle
            //matriceMapRectangles.push(rectangle)
            app.stage.addChild(rectangle);
          }
        }
      },
      /**
       * Fonction remplissant un réctangle dans le canvas de création
       * @param firstCase Case haut-gauche du rectangle
       * @param secondCase Case bas-droite du rectangle
       * @param caseType Eventuellement un type spécifique à remplir
       */
      fill(firstCase, secondCase, caseType = null) {
        // On détermine les cases les plus à gauche
        const iOrigin = Math.min(firstCase[0], secondCase[0]);
        const jOrigin = Math.min(firstCase[1], secondCase[1]);

        // Puis on récupère leur distance aux autres cases. Pour avoir un départ et une longueur & largeur de rectangle
        const iDistance = Math.abs(firstCase[0] - secondCase[0]);
        const jDistance = Math.abs(firstCase[1] - secondCase[1]);
        let newCase
        let couleur
        // Si aucun type n'est spécifié on récupère la couleur sélectionnée préalablement dans les data du component
        if(caseType === null) {
          couleur = this.color
          newCase = couleursInverse[this.color]
        } else {
          // Sinon on récupère la couleur associée au type demandé
          newCase = caseType
          couleur = couleurs[caseType]
        }

        // Puis on itère chaque case dans ce rectangle et on en change la couleur
        // Et on modifie son type dans la map de création
        for (let k = iOrigin; k <= iOrigin + iDistance; k++) {
          for (let l = jOrigin; l <= jOrigin + jDistance; l++) {
            console.log(couleur)
            matriceMapRectanglesFromCoord[`${k},${l}`].tint = couleur;
            this.$data.creationMap[l][k] = newCase;
          }
        }
      },
      /***
       * Fonction peuplant le CANVAS PixiJS de création de map
       *
       * Initialise une première fois chaque cas avec ses eventListeneur. Puis se contente de réafficher le canvas
       * d'affichage (en opposition avec celui de création)
       */
      drawPixiCreate() {
        const that = this
        const canvas = document.getElementById('pixi-create');
        const app = new PIXI.Application({
          width: width,
          height: height,
          antialias: true,
          transparent: false,
          view: canvas,
        })

        //let graphics = new PIXI.Graphics()
        //const wallTexture = PIXI.Sprite.from("wall.png");

        // On peuple la map de création avec des cases de couloirs (considérées comme cases neutres)
        this.createMap(that.$data.creationMap)

        // Pour x cases en longueur et largeur
        for (let i = 0; i < nbRect; i++) {
          for (let j = 0; j < nbRect; j++) {
            // On crée un sprite blanc que l'on va modifier pour représenter notre case
            let rectangle = new PIXI.Sprite(PIXI.Texture.WHITE);
            rectangle.eventMode = 'static';
            rectangle.cursor = 'pointer';

            // La première fois que le canvas de création est créé, on lui ajoute les eventlistener
            if(!initialized) {

              /**
               *  Fonction affichant sur la canvas une porte de métro tout en vérifiant qu'elle est bien positionnée.
               *  Ou alors s'il s'agit d'un autre type de case. Remplit l'aire correspondante par la couleur du type
               */
              function setColor() {
                // Dans le cas des portes de métro on les ajoute case par case
                if(couleursInverse[that.color] === "G") {
                  const voie = that.selectedPortesVoie === 1 ? that.voie1 : that.voie2
                  // On récupère le sens des rames
                  const sens = Math.abs(voie[0] - voie[2]) > Math.abs(voie[1] - voie[3])
                      ? "h"
                      : "v";

                  //On vérifie que la porte juxtapose une voie
                  //Cas ou la porte est au-dessus ou en dessous sur l'axe Y de la voie
                  //    []
                  // ======================
                  //              []
                  if((voie[0] <= i <= voie[2])
                      && ((j === voie[1] - 1) || (j === voie[3] + 1)))
                  {
                    // Si la voie va de gauche à droite
                    if(sens === "h") {
                      // La voie la plus proche du 0 est la première
                      if(that.selectedPortesVoie === 1) {
                        that.portesMetro[0].push([j,i])
                      } else {
                        that.portesMetro[1].push([j,i])
                      }
                      that.$data.creationMap[j][i] = "G";
                    } else {
                      that.texteErreur = "La porte doit être sur les côtés de sa voie";
                      return;
                    }
                    matriceMapRectanglesFromCoord[`${i},${j}`].tint = that.color;
                  }
                  //Cas ou la porte est à gauche ou droite sur l'axe X de la voie
                  // [] ||
                  //    || []
                  //    ||
                  else if((voie[1] <= j <= voie[3])
                      && ((i === voie[0] - 1) || (i === voie[2] + 1)))
                  {
                    if(sens === "v") {
                      // La voie la plus proche du 0 est la première
                      if(that.selectedPortesVoie === 1) {
                        that.portesMetro[0].push([j,i])
                      } else {
                        that.portesMetro[1].push([j,i])
                      }
                      that.$data.creationMap[j][i] = "G";
                    } else {
                      that.texteErreur = "La porte doit être sur les côtés de sa voie";
                      return;
                    }
                    matriceMapRectanglesFromCoord[`${i},${j}`].tint = that.color;
                  }
                  else
                  {
                      that.texteErreur = "La porte doit juxtaposer sa voie"
                  }
                  return;
                }


                if (selectCase === true) {
                  secondCase = [i, j];

                  switch (couleursInverse[that.color]) {
                    // Voies
                    case "Q":
                      let voie = that.selectedVoie === 1 ? that.voie1 : that.voie2

                      if(voie.length === 4) {
                        that.fill(
                            [voie[0], voie[1]],
                            [voie[2], voie[3]],
                            '_')
                      }
                      // On récupère le minimum pour pouvoir traiter les voies de haut en bas gauche à droite
                      voie[0] = min(firstCase[0],secondCase[0])
                      voie[1] = min(firstCase[1],secondCase[1])
                      voie[2] = max(firstCase[0],secondCase[0])
                      voie[3] = max(firstCase[1],secondCase[1])
                      break;
                  }
                  that.fill(firstCase, secondCase)
                  selectCase = false;
                } else {
                  firstCase = [i, j];
                  selectCase = true;
                }
              }

              rectangle.on("pointerdown", setColor)
              rectangle.on("mouseover", () => {
                if (rectangle.tint === 0xFFFFFF) rectangle.tint = 0xC0C0C0;
              })
              rectangle.on("mouseout", () => {
                if (rectangle.tint === 0xC0C0C0) rectangle.tint = 0xFFFFFF;
              })
            }

            rectangle.tint = 0xE9E9E9;
            rectangle.position.set(i * longRect, j * hautRect);
            rectangle.width = longRect;
            rectangle.height = hautRect;
            matriceMapRectanglesFromCoord[`${i},${j}`] = rectangle;
            app.stage.addChild(rectangle);
          }
        }
        //app.stage.addChild(graphics);
        initialized = true;
      },
      /**
       * Met à jour la couleur actuellement sélectionnée par une nouvelle
       * @param newColor Nouvelle couleur
       */
      setColor(newColor) {
        this.$data.color = newColor;
      },
      /**
       * Fonction remplissant une map de case vide (Case de couloirs)
       * @param map
       */
      createMap(map) {
        for (let j = 0; j < nbRect; j++) {
          let tmp = [];
          for (let i = 0; i < nbRect; i++) {
            tmp.push("_");
          }
          map[j] = tmp;
        }
      },
      /**
       * Fonction remplissant la map du mode affichage par la map temporaire de création
       */
      setNewMap() {
        for (let j = 0; j < nbRect; j++) {
          for (let i = 0; i < nbRect; i++) {
            this.map[j][i] = this.creationMap[j][i];
           // this.creationMap[i][j] = "_"
          }
        }
      },
      /**
       * Modifie l'affichage pour correspondre à l'état de création ou d'affichage
       */
      setModification() {
        // Affiche la roue de chargement
        document.getElementById("loader").style.display = "block";
        setTimeout(()=>{
          if(this.modification) {
            this.color = "0xE9E9E9"
            this.drawPixi();
            document.getElementById("change-mode").checked = false;
          } else {
            this.drawPixiCreate();
            document.getElementById("change-mode").checked = true;
          }
          this.modification = !this.modification;
        },1000)
        setTimeout(()=>{
          document.getElementById("loader").style.display = "none"},1000)
      },
      /**
       * Fonction récupérant les inputs des paramètres, puis les envoie au backend pour lancer la simulation
       */
      validateCreation() {
        this.setNewMap()
        this.params.map = this.creationMap
        this.params.leftTopCorners = [[this.voie1[1],this.voie1[0]],[this.voie2[1],this.voie2[0]]]
        this.params.rightDownCorners = [[this.voie1[3],this.voie1[2]],[this.voie2[3],this.voie2[2]]]
        this.params.gates = this.portesMetro
        this.params.goToLeft = [true, false]
        this.params.frequency = [5,5]
        this.params.stopTime = [5,5]
        this.params.capacity = [this.capacite, this.capacite]
        this.params.maxDuration = 500
        //Lancer simulation
        console.log(this.params.map)
        fetch("http://localhost:12000/configure", {
          method: "POST",
          mode:"no-cors",
          headers: {
            "Content-Type": "application/json"
          },
          body: JSON.stringify(this.params)
        })
        .then((res) => {
          document.getElementById("loader").style.display = "block";
          this.simulationAlive = true;
          setTimeout(()=> {
            this.setModification();
          },1000);
          setTimeout(() => {
            this.simulationLoop();
          },1000);
        })
        .catch(err => alert("Une erreur est survenue : \n"+err))
      },
      resetCreation() {
        this.etapeCreation = 0;
        this.drawPixiCreate();
      },
      nextEtape() {
        switch (this.etapeCreation) {
          case 0:
            this.etapeCreation++;
              this.texteErreur = "";
            break;
          case 1:
            if(this.voie1[0] !== undefined && this.voie2[0] !== undefined) {
              this.etapeCreation++
              this.texteErreur = "";
            } else {
              this.texteErreur = "Vous devez placer des voies"
            }
        }
      },
      simulationLoop() {
        setInterval(() => {
          if(this.simulationAlive) {
            fetch("http://localhost:12000/launch",
                {
                  method: "GET",
                })
                .then((res) => res.json())
                .then((data) => {
                  for (let i = 0; i < nbRect; i++) {
                    for (let j = 0; j < nbRect; j++) {
                      this.map[i][j] = data[j][i]
                      matriceMapRectanglesFromCoord[`${i},${j}`].tint = couleurs[data[j][i]]
                      // if(this.map[i][j] !== data[j][i]) {
                      //   this.map[i][j] = data[j][i]
                      //   console.log(this.map)
                      //     matriceMapRectanglesFromCoord[`${i},${j}`].tint = couleurs[data[i][j]]
                      // }
                    }
                  }
                })
              .catch((err) => console.log(err))
          } else {
            clearInterval()
          }
        },500)
      },
      passParamTab(mode) {
        if(mode === 0) {
          this.paramTab--
          if(this.paramTab === 0) this.paramTab = 2
        } else {
          this.paramTab++
          if(this.paramTab === 3) this.paramTab = 1
        }
      },

      test() {
        this.color = "0xE9E9E9"
        this.map = [["X", "X", "X", "X", "X", "X", "X", "X", "W", "W", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "W", "W", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "_", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "_", "X", "X", "X", "X", "_", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "_", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "_", "X", "X", "X", "X", "_", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "_", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "_", "_", "X", "X", "_", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "B", "B", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "B", "B", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "B", "B", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "B", "B", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "B", "B", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "B", "B", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "B", "B", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "Q", "B", "B", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "_", "_", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"],["X", "X", "X", "X", "S", "S", "X", "X", "X", "X", "X", "X", "E", "E", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "E", "E", "X", "X", "X", "X", "X", "X", "S", "S", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X", "X"]]
        this.params.map = this.map
        this.params.leftTopCorners = [[9,0],[11,0]]
        this.params.rightDownCorners = [[10,39],[12,39]]
        this.params.gates = [[[8, 5], [8, 6], [8, 34]],[[13, 5], [13, 6], [13, 34]]]
        this.params.goToLeft = [true,false]
        this.params.frequency = [5,5]
        this.params.stopTime = [5,5]
        this.params.capacity = [100,100]
        this.params.maxDuration = 150
        this.params.flow = 100
        this.params.controleurs = true
        this.params.fraudeurs = true
        this.params.impolis = true
        this.params.mob_reduite = true
        this.params.patients = true
        //Lancer simulation
        fetch("http://localhost:12000/configure", {
          method: "POST",
          mode:"no-cors",
          headers: {
            "Content-Type": "application/json"
          },
          body: JSON.stringify(this.params)
        })
        .then((res) => {
          document.getElementById("loader").style.display = "block";
          this.simulationAlive = true;
          setTimeout(()=> {
            this.setModification();
          },1000);
          this.simulationLoop();
          /*setTimeout(() => {

          },1000);*/
        })
        .catch(err => alert("Une erreur est survenue : \n"+err))
      },
    },
    mounted() {
      this.createMap(this.map)
      //this.drawPixi()
      this.drawPixiCreate()
    },
  }
</script>


<template>
  <div id="commandes-containers">
    <h2>Commandes</h2>
    <hr>
    <div id="commandes-switch">
      <div class="form-switch">
        <input class="form-check-input" type="checkbox" role="switch" id="change-mode" @click="setModification" checked>
        <label class="form-check-label" for="change-mode">
          <span v-show="modification === true">Création</span>
          <span v-show="modification === false">Simulation</span>
        </label>
        <button @click="test" class="btn btn-outline-success" style="margin-left:2vw" v-show="modification === true">Test</button>
      </div>
    </div>
    <hr>
    <div style="font-weight: bold;font-size: 2.5vh" v-show="modification === true">{{this.textEtapeCreation[this.etapeCreation]}}</div>
    <div class="commandes-liste" v-show="modification === true">
      <div v-show="this.etapeCreation === 0" class="option" v-on:click="setColor(this.$data.couleurs['X'])"><BoxOptions case="X"/><span>Mur</span></div>
      <div v-show="this.etapeCreation === 0" class="option" v-on:click="setColor(this.$data.couleurs['_'])"><BoxOptions case="_"/><span>Couloir</span></div>
      <div v-show="this.etapeCreation === 1" class="option" v-on:click=";this.selectedVoie=1;setColor(this.$data.couleurs['Q'])"><BoxOptions case="Q"/><span>Voie 1</span></div>
      <div v-show="this.etapeCreation === 1" class="option" v-on:click=";this.selectedVoie=2;setColor(this.$data.couleurs['Q'])"><BoxOptions case="Q"/><span>Voie 2</span></div>
      <div v-show="this.etapeCreation === 1" class="option" v-on:click="setColor(this.$data.couleurs['B'])"><BoxOptions case="B"/><span>Ponts</span></div>
      <div v-show="this.etapeCreation === 2" class="option" v-on:click="setColor(this.$data.couleurs['E'])"><BoxOptions case="E"/><span>Entrée</span></div>
      <div v-show="this.etapeCreation === 2" class="option" v-on:click="setColor(this.$data.couleurs['S'])"><BoxOptions case="S"/><span>Sortie</span></div>
      <div v-show="this.etapeCreation === 2" class="option" v-on:click="setColor(this.$data.couleurs['W'])"><BoxOptions case="W"/><span>Entrées et sorties</span></div>
      <div v-show="this.etapeCreation === 2" class="option" v-on:click=";this.selectedPortesVoie=1;setColor(this.$data.couleurs['G'])"><BoxOptions case="G"/><span>Portes voie 1</span></div>
      <div v-show="this.etapeCreation === 2" class="option" v-on:click=";this.selectedPortesVoie=2;setColor(this.$data.couleurs['G'])"><BoxOptions case="G"/><span>Portes voie 2</span></div>
    </div>
    <div style="font-weight: bold;font-size: 2.5vh" v-show="modification === false">Types d'agents</div>
    <div class="commandes-liste" v-show="modification === false">
      <div class="option"><BoxOptions case="C"/><span>Controleur</span></div>
      <div class="option"><BoxOptions case="N"/><span>Normal</span></div>
      <div class="option"><BoxOptions case="F"/><span>Fraudeur</span></div>
      <div class="option"><BoxOptions case="H"/><span>Mobilité réduite</span></div>
      <div class="option"><BoxOptions case="P"/><span>Patient</span></div>
    </div>
    <hr>
    <div v-show="this.texteErreur !== ''">
      {{this.texteErreur}}
      <hr>
    </div>
    <div v-if="modification === true">
      <button v-show="this.etapeCreation !== 2" class="btn btn-success" @click="nextEtape">Valider cette étape</button>
      <button v-show="this.etapeCreation === 2" class="btn btn-success" @click="validateCreation">Lancer la simulation</button>

      &nbsp;
      <button v-show="this.etapeCreation === 2" class="btn btn-danger" @click="resetCreation">Recommencer</button>
    </div>
    <hr>
    <div style="padding: 1vh 1vw 1vh 1vw">
      <form id="params" style="display: flex; flex-direction: row">
        <div v-show="this.paramTab === 1" class="form-row align-items-center">
          <!--           <div class="input-group mb-2">
                     <div class="input-group-prepend">
                        <div class="input-group-text">Durée max</div>
                      </div>
                      <input type="number" class="form-control" id="maxDuration" v-model="params['maxDuration']" placeholder="en secondes">
                    </div>-->
          <div class="input-group mb-2">
            <div class="input-group-prepend">
              <div class="input-group-text">Débit de création des agents</div>
            </div>
            <input type="number" class="form-control" id="flow" v-model="params['flow']" placeholder="1/ms">
          </div>
          <div style="display: flex;flex-direction: column;align-items: flex-start;text-align: left">
            <div class="form-switch">
              <input class="form-check-input" type="checkbox" role="switch" id="controleurs" v-model="params['controleurs']">
              <label class="form-check-label" for="controleurs" style="margin-left:1vw">Controleurs</label>
            </div>
            <div class="form-switch">
              <input class="form-check-input" type="checkbox" role="switch" id="fraudeurs" v-model="params['fraudeurs']">
              <label class="form-check-label" for="fraudeurs" style="margin-left:1vw">Fraudeurs</label>
            </div>
            <div class="form-switch">
              <input class="form-check-input" type="checkbox" role="switch" id="impolis" v-model="params['impolis']">
              <label class="form-check-label" for="impolis" style="margin-left:1vw">Impolis</label>
            </div>
            <div class="form-switch">
              <input class="form-check-input" type="checkbox" role="switch" id="mob_reduite" v-model="params['mob_reduite']">
              <label class="form-check-label" for="mob_reduite" style="margin-left:1vw;width:15vw">Mobilité réduite</label>
            </div>
            <div class="form-switch">
              <input class="form-check-input" type="checkbox" role="switch" id="patients" v-model="params['patients']">
              <label class="form-check-label" for="patients" style="margin-left:1vw;width:15vw">Présence d'usagers patients</label>
            </div>
          </div>
        </div>
        <div v-show="this.paramTab === 2" class="form-row align-items-center">
          <div class="input-group mb-2">
            <div class="input-group-prepend">
              <div class="input-group-text">Fréquence des trains</div>
            </div>
            <input type="number" class="form-control" id="frequency" v-model="frequence" placeholder="en secondes">
          </div>
          <div class="input-group mb-2">
            <div class="input-group-prepend">
              <div class="input-group-text">Temps d'arrêt</div>
            </div>
            <input type="number" class="form-control" id="stopTime" v-model="stop" placeholder="en secondes">
          </div>
          <div class="input-group mb-2">
            <div class="input-group-prepend">
              <div class="input-group-text">Capacité</div>
            </div>
            <input type="number" class="form-control" id="capacity" v-model="capacite">
          </div>
        </div>
      </form>
    </div>
    <div id="commmandes-end-container">
      <div id="arrow-container">
        <button class="btn btn-outline-secondary" @click="passParamTab(0)">
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-arrow-left" viewBox="0 0 16 16">
            <path fill-rule="evenodd" d="M15 8a.5.5 0 0 0-.5-.5H2.707l3.147-3.146a.5.5 0 1 0-.708-.708l-4 4a.5.5 0 0 0 0 .708l4 4a.5.5 0 0 0 .708-.708L2.707 8.5H14.5A.5.5 0 0 0 15 8"/>
          </svg>
        </button>
        &nbsp;
        <button class="btn btn-outline-secondary" @click="passParamTab(1)">
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-arrow-right" viewBox="0 0 16 16">
            <path fill-rule="evenodd" d="M1 8a.5.5 0 0 1 .5-.5h11.793l-3.147-3.146a.5.5 0 0 1 .708-.708l4 4a.5.5 0 0 1 0 .708l-4 4a.5.5 0 0 1-.708-.708L13.293 8.5H1.5A.5.5 0 0 1 1 8"/>
          </svg>
        </button>
      </div>
      <div style="font-size: 0.8em;margin-top: 1vh"><i>{{this.paramTab}} / 2</i></div>
    </div>
  </div>

<!--  <button class="btn btn-danger" @click="this.simulationAlive = false">Stopper</button>-->
  <div class="map-container" ref="ref">
    <canvas id="pixi" v-show="this.modification === false"></canvas>
    <canvas id="pixi-create" v-show="this.modification === true"></canvas>
<!--    style="display: none"-->
  </div>
  <div class="loader" id="loader">
    <div class="loader-inner">
      <div class="loader-line-wrap">
        <div class="loader-line"></div>
      </div>
      <div class="loader-line-wrap">
        <div class="loader-line"></div>
      </div>
      <div class="loader-line-wrap">
        <div class="loader-line"></div>
      </div>
      <div class="loader-line-wrap">
        <div class="loader-line"></div>
      </div>
      <div class="loader-line-wrap">
        <div class="loader-line"></div>
      </div>
    </div>
  </div>
</template>

<style scoped>
  #commmandes-end-container {
    height: 10vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding-bottom: 1vh;
  }
  #arrow-container {
    height: 16vh;
    display: flex;
    flex-direction: row;
    align-items: flex-end;
    gap: 5px;
    justify-content: center;
  }
  #pixi-create {
    border: solid black 1px;
  }
  .loader {
    bottom: 0;
    left: 0;
    overflow: hidden;
    position: fixed;
    right: 0;
    top: 0;
    z-index: 99999;
    display: none;
  }

  .loader-inner {
    bottom: 0;
    height: 60px;
    left: 0;
    margin: auto;
    position: absolute;
    right: 0;
    top: 0;
    width: 100px;
  }

  .loader-line-wrap {
    animation:
        spin 2000ms cubic-bezier(.175, .885, .32, 1.275) infinite;
    box-sizing: border-box;
    height: 50px;
    left: 0;
    overflow: hidden;
    position: absolute;
    top: 0;
    transform-origin: 50% 100%;
    width: 100px;
  }
  .loader-line {
    border: 4px solid transparent;
    border-radius: 100%;
    box-sizing: border-box;
    height: 100px;
    left: 0;
    margin: 0 auto;
    position: absolute;
    right: 0;
    top: 0;
    width: 100px;
  }
  .loader-line-wrap:nth-child(1) { animation-delay: -50ms; }
  .loader-line-wrap:nth-child(2) { animation-delay: -100ms; }
  .loader-line-wrap:nth-child(3) { animation-delay: -150ms; }
  .loader-line-wrap:nth-child(4) { animation-delay: -200ms; }
  .loader-line-wrap:nth-child(5) { animation-delay: -250ms; }

  .loader-line-wrap:nth-child(1) .loader-line {
    border-color: hsl(0, 80%, 60%);
    height: 90px;
    width: 90px;
    top: 7px;
  }
  .loader-line-wrap:nth-child(2) .loader-line {
    border-color: hsl(60, 80%, 60%);
    height: 76px;
    width: 76px;
    top: 14px;
  }
  .loader-line-wrap:nth-child(3) .loader-line {
    border-color: hsl(120, 80%, 60%);
    height: 62px;
    width: 62px;
    top: 21px;
  }
  .loader-line-wrap:nth-child(4) .loader-line {
    border-color: hsl(180, 80%, 60%);
    height: 48px;
    width: 48px;
    top: 28px;
  }
  .loader-line-wrap:nth-child(5) .loader-line {
    border-color: hsl(240, 80%, 60%);
    height: 34px;
    width: 34px;
    top: 35px;
  }

  @keyframes spin {
    0%, 15% {
      transform: rotate(0);
    }
    100% {
      transform: rotate(360deg);
    }
  }
  #commandes-containers {
    padding-top: 1vh;
    height: inherit;
    width: 30vw;
    border: solid black 1px;
  }
  #commandes-switch {
    font-size: 3.5vh;
    display: flex;
    flex-direction: row;
    justify-content: center;
  }

  .commandes-liste {
    font-size: 3.5vh;
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
    justify-content: space-evenly;
    /*flex-direction: column;*/
    align-items: flex-start;
    padding: 1vw 1vh 1vw 1vh;
    /*padding: 2vh 6vw 2vh 35%;*/
  }

  #change-mode {
    margin-right: 3vw;
  }

  .option {
    padding: 1vh 1vw 1vh 1vw;
    display: flex;
    flex-direction: row;
    align-items: center;
  }
  .option:hover {
    background-color: #cecece;
  }

</style>