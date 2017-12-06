+++
date = "2017-02-12T22:05:09+01:00"
title = "Migrer de Webpack vers FuseBox (v1)"
draft = true
+++

## Contexte

Nous utilisons actuellement Webpack pour compiler les assets de l'application mais le résultat n'est pas complétement satisfaisant. Le temps de compilation initial est de 30s mais il fut d'environ 60s avant d'avoir fait un ensemble d'optimisations :

- Utilisation de caches (babel, iconfont), 
- Split de bundle (DLLPlugin plutôt que CommonChunkPlugin) 
- Désactivation de source maps

En utilisant le Dev Server on a un temps de recompilation relativement long de 2s en moyenne. Ce n'est sans doute pas au niveau du temps de recompilation que la différence va se faire entre les deux outils. 
La question est donc de savoir s'il est possible de faire mieux malgré les efforts déjà réalisés avec FuseBox. La promesse des developpeurs : simplicité et performance.

## Le poc

J'ai fait un petit projet juste pour tester la compatiblité de Webpack avec le [React Hot Loader de gaearon](https://github.com/gaearon/react-hot-loader). Il permet de valider que HMR fonctionne bien avec les composants, styles et reducers. Il n'y a que 3 composants mais il fait le travail. Installer FuseBox sur ce projet est vraiment rapide, en moins d'une heure vous avec un build de dev qui fonctionne (malgré la documentation avare en explications). J'ai repris l'exemple de la documentation officielle et y ai ajouté quelques paramètres. Voici le fichier `fuse.js` complet : 

```javascript
const fsbx = require("fuse-box");

// Récupération de la config d'environnement via DotEnv
const dotenv = require('dotenv')
const env = dotenv.config().parsed // retourne un object

// Création d'une instance FuseBox
const fuseBox = new fsbx.FuseBox({
    // Code base root path of your app
    homeDir: "src/",
    // Fichier JS de sortie de l'application à inclure dans le HTML
    outFile: "./build/bundle.js",
    // Configure source maps
    sourceMap: {
        bundleReference: "sourcemaps.js.map",
        outFile: "./build/sourcemaps.js.map",
    },
    // Plugins pour gérer les différents types de dépendences
    plugins: [
    	 // Fichier SCSS pour générer un fichier de styles
    	 // Ici, on a un groupe de plugins qui vont traiter le même type de fichier
        [
            fsbx.SassPlugin(),
            fsbx.CSSResourcePlugin({ inline: true }),
            fsbx.CSSPlugin()
        ],
        // Injection de variables d'environnement avec l'aide de DotEnv
        fsbx.EnvPlugin(env),
        // Babel pour la transpilation des fichier ES2015 et JSX
        fsbx.BabelPlugin({
            config: {
                sourceMaps: true,
                presets: [["es2015", { "loose": true }], "react"],
                plugins: [
                    "react-hot-loader/babel",
                    "transform-class-properties"
                ]
            }
        }),
        fsbx.UglifyJSPlugin(),
        fsbx.WebIndexPlugin({
            target : '../index.html'
        })
    ]
});

// Lancement du dev server en indiquant le point d'entrée de notre application
// Notez le chevron qui indique que l'on veut compiler tout sans les dépendences
// Pus d'info : http://fuse-box.org/#bundle
fuseBox.devServer("> index.js");
```

TODO explication fusebox, plugins/loadrs, config bundle

La différence est assez mineure : 

- Compilation en prod :
  - **FuseBox** : 6-7s (383Ko)
  - **Webpack** : 6-7s (309K)

- Re-compilation en mode dev :
  - **FuseBox** : 500ms 
  - **Webpack** : 2s
    Dans les 2 cas, une 2e modification sur le fichier coute ~150ms

_Les temps sont une moyenne de plusieurs lancements._

## Cas de la vraie vie

On a prouvé que c'est rapide sur un petit projet très basique mais qu'en est-il de l'utilisation de FuseBox sur un 
véritable projet conséquant déjà mis en production. 
Je vais prendre l'exemple de notre application pour laquelle on doit intégrer les 
fonctionnalités suivantes dans la chaîne de build. De manière simple FuseBox nous permet de gérer 
toutes ces points avec les plugins embarqués à l'exception des fonts, favicons, le fichier index.html et Eslint.

Voici en détail les différents de la génération du bundle

##### React/Redux avec HMR

Faisable avec un plugin embarqué.
Fonctionne mais je perd l'état du Redux store reload.

##### Transpilation ES6/JSX avec Babel

Faisable avec un plugin embarqué.
La gestion du `.babelrc` est disponible mais est en cours d'amélioration. Je n'ai pas encore eu de probléme de coté là.

##### Compiler 2 fichiers JS en sortie (vendors et app)

L'idée ici est de faire un bundle pour les vendors et un autre pour l'application. 
Les vendors ne changeant pas souvent, seront regulierement servis à partir du cache (à condition de bien gérer les headers)
Il est théoriquement possible de le faire avec Fusebox mais je n'ai pas encore essayé.

TODO

##### SASS

Faisable avec un plugin embarqué
Fonctionne, attention à bien spécifier `includePaths' pour les ressources appelées depuis le SASS

Exemple : 

```
fsbx.SassPlugin({
    includePaths: [
        path.resolve('./src/sass'),
        path.resolve('./src/images')
    ]
}),
```
##### Eslint

Utilisation d'un plugin tiers, attention à bien chainer le plugin Eslint avec Babel pouréviter un problème de parsing. 
 
```
const fsbx = require("fuse-box");
const eslinter = require("fuse-box-eslint-plugin");
const fuseBox = new fsbx.FuseBox({
    // ...
    plugins: [
        [
            eslinter({
                pattern: /js(x)*$/,
            }),
            fsbx.BabelPlugin(),
        ]
    ]
});
```

Ne fonctionne pas avec la derniere version de Fusebox. Le plugin est référence sur le site mais le dépôt source est très peu actif.

##### DotEnv configuration

Faisable avec un plugin embarqué. Il suffit de require le package qui va parser le contenu du fichier, puis passer le résultat à la config FuseBox via le plugin: 

```js
const dotenv = require('dotenv')
const env = dotenv.config().parsed

{
    //...
    plugins : [
        fsbx.EnvPlugin(env),
    ]
    //...
}    
```
 
##### Import d'images

Faisable avec un plugin qui ebarque les images en Base64, ce qui peut alourdir considérablement le poids du bundle.
L'import des images (et autres assets) fait depuis les fichiers JS doivent pointer vers un emplacement à l'intérieur du `homeDir` de fuse.

##### Webfont (Roboto)

Rien de particulier ici, il suffit d'utiliser la méthode `@import` dans le sass pour importer notre police Roboto.

##### Géneration d'une iconfont depuis des SVG

Pour optimiser le poids des ressources et pouvoir personnaliser au maximum nos icones, nous avons fait le choix de générer notre propre police d'icones. Cela se fait à partir d'un ensemble de fichier SVG et grâce à un plugin de Webpack.

Pour FuseBox il n'existe pas d'alternative à cette solution. 
Donc il faut regénérer la police manuellement ([Fontello](http://fontello.com), [Icomoon](https://icomoon.io)) 
et l'inclure avec un import.


##### Géneration du fichier index.html racine

Webpack permet, via un plugin de générer un fichier HTML à partir d'un template au format `.ejs` et de l'ajouter au bundle
Ce fichier racine servira de conteneur à l'app JS pour le navigateur. Pratique pour injecter des données dynamiques (comme injecter les ressources types images, favicon et autres).
Coté FuseBox, un [plugin](http://fuse-box.org/plugins/web-index-plugin) (moins configurable)  existe pour faire ça.

##### Favicon

Nous utilisions un plugin pour générer un ensemble de favicons pour tous les devices à partir d'un seul fichier SVG. 
Le plugin s'occupait de la générations des images/ico puis ajoutait à notre fichier HTML (`.ejs`) les balises et manifestes nécessaires pour dans le `<head>`. Impossible encore avec FuseBox.


## Promesse tenue

L'équipe de FuseBox se concentre vraiment sur les performances et ça se voit, elles sont au rendez-vous, avec la simplicité de la configuration en prime. Cepandant, étant donné sa relative jeunesse, certaines choses m'ont manqué (gestion des images) mais j'ai pu les contourner en faisant différement. 
Concernant le poids du bundle généré c'est Webpack qui s'en sort mieux : 383Ko pour FuseBox et 325Ko pour Webpack.

Avantages 
- Léger gain performance en dev
- Simplicité de configuration

Inconvénients : 
- Typescript en peer dep même si je ne m'en sers pas
- HMR non fonctionnel avec le store Redux
- Uglification assez longue (plus rapide avec Webpack) 
- Fichiers généré plus lourd
