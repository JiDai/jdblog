+++
date = "2017-02-12T22:05:09+01:00"
title = "FuseBox (v2) en tant qu'alternative à Webpack (v3)"
draft = true
+++

## Contexte

Nous utilisons actuellement Webpack pour compiler les assets de l'application mais le résultat n'est pas complétement satisfaisant. Le temps de compilation initial est de 30s mais il fut d'environ 60s avant d'avoir fait un ensemble d'optimisations :

- Utilisation de caches (babel, iconfont), 
- Split de bundle (DLLPlugin plutôt que CommonChunkPlugin) 
- Désactivation de source maps

En utilisant le Dev Server on a un temps de recompilation relativement long de 2s en moyenne. Ce n'est sans doute pas au niveau du temps de recompilation que la différence va se faire entre les deux outils. 
La question est donc de savoir s'il est possible de faire mieux, malgré les efforts déjà réalisés, avec FuseBox. La promesse des développeurs : simplicité et performance.

FuseBox est donc, comme Webpack, un "bundler". Depuis un point d'entrée (typiquement `index.js`), le bundler parcours le code de l'application afin de trouver les directives d'import, ajouter les dépendances correspondantes au bundle puis copier ce fichier de bundle dans un dossier de sortie spécifié. Les dépendances peuvent être des fichiers JS evidemment mais aussi des fichiers CSS (SASS/LESS) images, JSON... Chaque fichier ajouté au bundle est traité avec un plugin (loader pour Webpack) qui va traduire le code en JS (et faire une copie par exemple pour les images dans le dossier de sortie).

## Example d'utilisation

J'ai fait un petit projet juste pour tester la compatiblité de Webpack avec le [React Hot Loader de gaearon](https://github.com/gaearon/react-hot-loader). Il permet de valider que HMR fonctionne bien avec les composants, styles et reducers. Il n'y a que 3 composants mais il fait le travail. Installer FuseBox sur ce projet est vraiment rapide, en moins d'une heure vous avec un build de dev qui fonctionne. J'ai repris l'exemple de la documentation officielle et y ai ajouté quelques paramètres. Voici le fichier `fuse.js` complet pour l'environnement de production:

```javascript
const fsbx = require("fuse-box");

// Récupération de la config d'environnement via DotEnv
const dotenv = require('dotenv')
const env = dotenv.config().parsed // retourne un Object

// Création d'une instance FuseBox
const fuse = new fsbx.FuseBox({
    // Code base root path of your app
    homeDir: "src/",
    // Fichier JS de sortie de l'application à inclure dans le HTML
    outFile: "./build/$name.js",
    // Désactive source maps
    sourcemaps: false,
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
        // Utilise le `.babelrc` et surchage les paramètres suivants :
        fsbx.BabelPlugin({
            sourceMaps: false,
        }),
        fsbx.UglifyJSPlugin(),
        // Créé un fichier `index.html` à la racine du build pour contenir l'application
        fsbx.WebIndexPlugin({
            target : '../index.html'
        })
    ]
});

// Créée le bundle de vendors
fuse.bundle("app")
    .instructions("> index.js")

// Lance la compilation
fuse.run()
```

Pour le mode dev, la configuration est sensiblement la même sauf que l'on va spécifier comment lancer un serveur local pour le reload/HMR de nos resources :

```
fuse.dev({}, server => {
    const app = server.httpServer.app;
    app.use(express.static(path.join('public')));
    app.get("*", function (req, res) {
        res.sendFile(path.resolve('public', 'index.html'));
    });
});
fuse.run();
```

Au final avec cette configuration la différence est assez mineure :

- Compilation en prod :
  - **FuseBox** : 5-6s (340Ko)
  - **Webpack** : 6-7s (300K)

- Re-compilation en mode dev :
  - **FuseBox** : 500ms
  - **Webpack** : 1.5s
    Dans les 2 cas, une 2e modification sur le fichier coûute ~150ms

_Les temps sont une moyenne de plusieurs lancements._

Concernant le poids du bundle généré c'est Webpack qui s'en sort mieux.

## Cas de la vraie vie

On a prouvé que c'est rapide sur un petit projet très basique mais qu'en est-il de l'utilisation de FuseBox sur un véritable projet conséquant déjà mis en production.
Je vais prendre l'exemple de notre application pour laquelle on doit intégrer les fonctionnalités suivantes dans la chaîne de build. De manière simple FuseBox nous permet de gérer quasiment toutes ces points avec les plugins embarqués.

En détail, les différents fonctionnalités de la génération du bundle :

##### React/Redux avec HMR

Faisable en natif.
Le HMR fonctionne mais je perd l'état du Redux store reload, chose qui n'arrivait pas avec Webpack. Dommage, je n'ai pas réussi à le faire fonctionner.

##### Transpilation ES6/JSX avec Babel

Faisable avec un [plugin](http://fuse-box.org/plugins/babel-plugin) embarqué. Aucun souci de ce coté là.

##### Compiler 2 fichiers JS en sortie (vendors et app)

L'idée ici est de faire un bundle pour les vendors et un autre pour l'application. 
Les vendors ne changeant pas souvent, ils seront régulierement servis à partir du cache (à condition de bien gérer les headers)
Faisable très facilement maintenant avec la version 2 de FuseBox : un exemple est disponible dans la doc et fonctionne directement dans mon cas :

```javascript
fuse.bundle("vendor")
    // Surveille les fichiers
    .watch()
    // Le premier bundle contient le code nécessaire à l'execution du HMR
    .hmr()
    // Le tilde indique que l'on veut les dépendences qui ne sont pas dans l'app (soit les node_modules)
    .instructions(" ~ index.js") // tilde is for get deps, except project deps

fuse.bundle("app")
    .watch()
    // Ce bundle ne contient donc pas le code du HMR (contrairement au premier)
    // mais on peut l'activer sur ce bundle
    .hmr()
    // Active les sourcemaps du package
    .sourceMaps(true)
    // Les crochets indique qu'on exclus les dépendences (elles sont dans le bundle `vendor`)
    // Le ! est pour retirer l'API de FuseBox
    .instructions(" !> [index.js]")
```

Le reste de la configuration ne change pas. Le détail des instructions [se trouve là](http://fuse-box.org/page/bundle#arithmetic-symbols)

##### SASS

Faisable avec un [plugin](http://fuse-box.org/plugins/sass-plugin) embarqué. Attention à bien spécifier `includePaths' pour les ressources appelées depuis le SASS

Exemple : 

```
plugins: [
    //...
    fsbx.SassPlugin({
        includePaths: [
            path.resolve('./src/sass'),
            path.resolve('./src/images')
        ]
    }),
]
```

Mais le SASS tout seul ne suffit pas, il va retourner du CSS qui ne va pas être compréhensible par l'appication. Il faut donc le chainer avec le plugin CSS pour que le tout communique bien :

```
plugins: [
    //...
    [
        fsbx.SassPlugin({
            includePaths: [
                path.resolve('./src/sass'),
                path.resolve('./src/images')
            ]
        }),
        fsbx.CSSPlugin()
    ]
]
```

##### Eslint

Utilisation d'un [plugin](https://github.com/DoumanAsh/fuse-box-eslint-plugin) tiers, attention à bien chainer le plugin Eslint avec Babel pour éviter un problème de parsing.
 
```
    plugins: [
        // ...
        [
            eslinter({
                pattern: /js(x)*$/,
            }),
            fsbx.BabelPlugin(),
        ]
    ]
});
```

Cepandant, l'exemple fonctionne avec la version 1 de Fuse-box mais plus avec la dernière. Le plugin est référencé sur le site officiel mais le dépôt source est très peu actif.

##### DotEnv configuration

Faisable avec un [plugin](http://fuse-box.org/plugins/env-plugin embarqué. Il suffit de require le package qui va parser le contenu du fichier, puis passer le résultat à la config FuseBox via le plugin:

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

Faisable avec un [plugin](http://fuse-box.org/plugins/image-base64-plugin qui embarque les images en Base64, ce qui peut alourdir considérablement le poids du bundle.
L'import des images (et autres assets) appelés depuis les fichiers JS doivent pointer vers un emplacement à l'intérieur du `homeDir` de fuse.

##### Webfont (Roboto)

Rien de particulier ici, il suffit d'utiliser la méthode `@import` dans le sass pour importer notre police Roboto.

##### Géneration d'une iconfont depuis des SVG

Pour optimiser le poids des ressources et pouvoir personnaliser au maximum nos icones, nous avons fait le choix de générer notre propre police d'icônes. Cela se fait à partir d'un ensemble de fichier SVG et grâce à un plugin de Webpack.
Pour FuseBox, il n'existe pas d'alternative à cette solution. Il faut générer la police manuellement (via [Fontello](http://fontello.com), [Icomoon](https://icomoon.io) par exemple) et l'inclure avec un import.

##### Géneration du fichier index.html racine

Webpack permet, via un plugin de générer un fichier HTML à partir d'un template au format `.ejs` et de l'ajouter au bundle
Ce fichier racine servira de conteneur à l'app JS pour le navigateur. Pratique pour injecter des données dynamiques (comme injecter les ressources types images, favicon et autres).
Coté FuseBox, un [plugin](http://fuse-box.org/plugins/web-index-plugin) existe pour faire ça. Il reste cepandant moins configurable

##### Favicon

Nous utilisions un plugin pour générer un ensemble de favicons pour tous les devices à partir d'un seul fichier SVG. 
Le plugin s'occupait de la générations des images/ico puis ajoutait à notre fichier HTML (`.ejs`) les balises et manifestes nécessaires pour dans le `<head>`. Impossible encore avec FuseBox.
Cette solution de génération automatique est quelque peu surdimensionnée, Mais les favicon ne changent pas souvent, on peut gérer ca manuellement.


## Promesse tenue ?

L'équipe de FuseBox se concentre vraiment sur les performances mais je suis un peu déçu du résultat "out-of-the-box", mais la simplicité est là : la configuration est simple et lisible.
De plus, étant donné sa relative jeunesse, certaines choses m'ont manqué (gestion des images) que j'ai pu les contourner en faisant manuellement.

Avantages
- Léger gain performance par raport a Webpack
- Simplicité de configuration
- Plugins embarqués pour faire 90% des tâches habituelles

Inconvénients :
- Typescript en peer dep même si je ne m'en sers pas
- HMR non fonctionnel avec le store Redux
- Uglification assez longue (plus rapide avec Webpack)
- Fichiers généré plus lourd

A l'heure où des outils comme `create-react-app` ou encore [VitaminJS](https://github.com/Evaneos/vitaminjs) permettent de se premunir de cette configuration manuelle, FuseBox continue sur la même voie que Webpack en la simplifiant tout en laissant la main au développeur.
Si vous cherchez une autre alternative essayez [Parcel](https://github.com/parcel-bundler/parcel)
