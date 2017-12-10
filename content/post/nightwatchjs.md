---
title: "NightWatchJS pour tester vos évènements Google Analytics"
date: 2017-12-10T20:28:50+01:00
draft: true
---

### 

Vous pourrez apprendre en lisant cet article comment implémenter un
environnement de test fonctionnels avec
[NightWatchJS](http://nightwatchjs.org/). Docker nous servira pour contenir le
serveur Selenium de test, ainsi l’utiliser facilement sans installer *Java* ni
*Selenium Server* sur la machine. Le code complet de l’application est
disponible sur [Github](https://github.com/JiDai/e2e-example). <br> Le site demo
(présent dans le dossier `testing-website`) qui sera testé est disponible
[ici](https://e2e-example.netlify.com/).

Tout n’est pas détaillé sur cet article, n’hésitez à parcourir le code.

Pour le tutorial, l’application de démo se concentre uniquement de tester les
envois à *Google Analytics*. Elle permet de valider en permanence que le plan de
tagging est toujours fonctionnel, chose importante quand les décisions produit
se basent sur ces chiffres. 

**Prérequis :**

* [Docker](https://www.docker.com/docker-mac) installé
* [Node 8](https://nodejs.org/en/download/)
* Connaissances des bases de NighWatchJS

Le projet a été testé sous Mac OSX mais devrait fonctionner sur un Windows ou
Linux.

### Préparation des dépendances


NPM va installer 2 dépendances : `nightwatch` et `better-npm-run`. Cette
dernière nous sert juste à injecter facilement les variables d’environnement
dans l’application.

### Selenium

Docker nous permet d’abstraire le serveur *Selenium*. Donc pour lancer le
serveur lancez la commande `docker-compose -p selenium up -d` et le serveur est
prêt. Vous pouvez le vérifier en lancant un navigateur sur l’URL
`http://localhost:4444/grid/console`. Vous devriez avoir une page ressemlant à
ca :

![](/images/selenium-grid.png)

L’image Docker contient deux instance de navigateur : *Chrome* et *Firefox.
*Dans notre exemple nous n’utiliserons que *Chrome.*

Enfin, il ne reste plus qu’à lancer `npm test` pour executer la suite de tests
et voir la sortie suivante si tout va bien : 

![](/images/nightwatch-ouput.png)

Vous pouvez aussi spécifier une suite et/ou un test avec la commande


### Travailler avec NightWatchJS

### Configuration

Tout se passe dans le fichier `nightwatch.config.js`. Je ne vais pas rentrer
dans le détail ici, si vous voulez en savoir plus, le fichier du projet est
assez commenté pour mieux comprendre le comportement de chaque option.

### PageObjects

Une bonne pratique recommandée par les développeurs de Selenium est d’utiliser
des *PageObjects*. Ce sont des objets qui vont décrire la page sur laquelle on
veut interagir. Dans *NighWatchJS*, les pages sont automatiquement charger dans
l’application pour chaque fichier ajouté dans le dossier `pages` (défini dans la
[configuration](https://github.com/JiDai/e2e-example/blob/master/nightwatch.config.js#L15)).
Le nom du fichier servira au nom de la propriété de `browser.page`dans le code.

[Example](https://github.com/JiDai/e2e-example/blob/master/pages/HomePage.js#L4)
dans l’application : Pour un fichier `pages/HomePage.js` le PageObject est
instanciable via `browser.page.HomePage()`

Dans la page, j’ai spécifié 2 élements internes à la page d’accueil :  les 2
boutons sur lesquels je vais simuler des clics. Pour simuler un clic sur le
bouton ‘Purchase’ puis le bouton ‘Contact us’ :


**Note :** Petit avertissement sur la méthode `click`. Le clic sur un élement ne
peut se faire que sur un élement réellement visible. Si l’élément est seulement
présent dans le DOM, cela ne fonctionnera pas. Il vaut donc mieux vérifier qu’il
l’est avant : 

    Home.waitForElementVisible('@purchaseCta', 5000, true);

Si vous testez en mobile et desktop, cela eut devenir un enfer selon la
complexité de vos media queries. N’hésitez pas alors à créer des scénarios
différents.

*****

Les PageObjects peuvent être paraître complexes au départ mais cela permet de
rendre le code des tests plus lisible et de maintenir un référentiel exhaustif
des élements testés à un seul endroit et donc de faciliter les éventuels
changements.

### Custom assertions et command

Une autre fonctionnalité de *NightWatchJS* qui simplifira l’écriture de test est
la possibilité d’étendre l’API avec l’ajout de commandes et assertions
personnalisées. Une commande permet de communiquer avec le navigateur, soit pour
lui envoyer des inscrutions soit pour récupérer des informations. Une assertion
est la même notion que dans n’importe quel outil de test.

Comme les *PageObjects*, pour ajouter une commande ou assertion personnalisée,
il suffit de créer un fichier dans le dossier approprié (et défini dans la
[configuration](https://github.com/JiDai/e2e-example/blob/master/nightwatch.config.js#L19)).
Le nom du fichier servira donc à la création d’une propriété sur
`browser.assert` ou `browser.verify` pour les assertions. Les commandes seront
définies directement sur l’objet `browser`.

Sa création est assez simple
([exemple](https://github.com/JiDai/e2e-example/blob/master/commands/scrollToElement.js)).
Pour définir une commande, il faut exporter une méthode `command` dans votre
fichier :

* Toujours retourner `browser` pour permettre le chaînage.
* Dans le cas de commande avec du code asynchrone, il est bon d’ajouter un
callback avec un argument`callback` à votre commande, puis appelez
`callback.call(browser)`.

Une assertion est plus complexe mais le principe est le même. Il faut exporter
une fonction `assertion` qui doit implémenter 4 membres
([exemple](https://github.com/JiDai/e2e-example/blob/master/assertions/ga.js)) :

* `expected` Représente la valeur attendue
* `value` Fonction qui va retourner la valeur utilisée dans l’assertion à comparer
avec `expected`
* `pass` Fonction qui exécute le test de l’assertion, doit retourner un boolean.
* `command` Commande qui va être exécutée pour récupérer la valeur à tester. Le
callback de cette fonction sera utilisé en argument de `value`

Vous pourrez trouver d’autres exemples d’assertions et commandes:

* [https://github.com/mobify/nightwatch-commands/tree/develop/commands](https://github.com/mobify/nightwatch-commands/tree/develop/commands)
* [https://github.com/mobify/nightwatch-commands/tree/develop/assertions](https://github.com/mobify/nightwatch-commands/tree/develop/assertions)
* [https://github.com/maxgalbu/nightwatch-custom-commands-assertions/](https://github.com/maxgalbu/nightwatch-custom-commands-assertions/)
* [https://github.com/yyx990803/nightwatch-helpers](https://github.com/yyx990803/nightwatch-helpers)

Dans le projet, j’ai créé une assertion pour verifier qu’un évènement *Google
Analytics* est bien envoyé aprés une action quelconque. La commande de cette
assertion va scruter le `localStorage` pour trouver les entrées préalablement
remplies par une action (un clic sur un bouton par ex.). L’assertion va de pair
avec la
[commande](https://github.com/JiDai/e2e-example/blob/master/commands/mockGA.js)
qui va injecter un script dans la page pour remplacer la fonction `ga()` par
défaut de Google Analytics. C’est cette fonction qui va remplir le
`localStorage` de tous les évènement envoyés.

**Note :** Vous remarquerez que l’assertion est vraie si on n’est pas sur
*Chrome*. Nous testons notre plan de tagging que sur *Chrome*, car le temps
d’éxecution total serait trop long.

### Next steps

* Il est assez simple de brancher [BrowserStack](https://www.browserstack.com/)
avec *NightWatchJS* pour tester beaucoup plus de navigateurs.
* Pour de l’intégration continue avec *Jenkins*, un `Dockerfile` d’exemple est
disponible dans l’app.
* Test de non-regression visuelle avec
[ResembleJS](https://www.npmjs.com/package/resemblejs)

*****

NightWatchJS est donc un outil formidable pour implémenter vos tests
fonctionnels. Mais la route est longue avant d’avoir un environement de test
complet et stable pour tous les navigateurs. Si vous avez déjà intégré des pages
web compatibles pour tous les navigateurs, vous devinez que vous devrez faire
face à des problèmes équivalents lors de l’écriture de test fonctionnels.
Certains navigateurs sont plus capricieux que d’autres, il y a toujours des
petites surprises insoupçonnées.<br> Il faut ensuite penser à la maintenabilité
des sceniaros testés, le temps d’exécution (notemment  sur BrowserStack).
