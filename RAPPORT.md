# Rapport PDC Texte

Anthony ROSSI, Yizhou TANG

## Travail effectué

Suite à la première itération, nous avons décidé d'orienter notre travail selon différents axes. 

Tout d'abord, nous avons réalisé une première version basique de l'index, entièrement en memoire, avec un système de requêtes simple (conjonctive ou disjonctive, algorithme naif).

Ensuite, nous avons ajouté un parser + lexer de requête. Ainsi, nous pouvons effectuer des requêtes plus complexes. Les symboles reconnus sont les parenthèses, l'opérateur or, et l'opérateur and (par défaut si aucun opérateur n'est placé entre les termes).
Un exemple de suite de requêtes permettant de tester le système est :
dune and (herbert or sahara)
dune and herbert
dune and sahara
dune and herbert or sahara

Afin de gagner du temps, nous avons mis en place un système de sauvegarde de l'index "naif", en utilisant pickle pour sauvegarder directement les objets manipulés. 
Ensuite, afin de faciliter le traitement des données, et d'accelerer les indexations ultérieures, nous avons remis en forme les données, en utilisant le formalisme JSON.

Nous avons ensuite transféré l'index vers un fichier sur disque (memory mapped file), avec un index en memoire ne contenant que les termes, et l'offset dans les fichiers d'index (triés par ordre alphabétique des termes).
Les fichiers, en effet, il existe deux fichiers : un fichier dont les entrées (pour chaque terme) sont triées par ID de document, l'autre dont les entrées sont triées par score.
En pratique, seul le fichier trié par ID est utilisé, l'autre est présent en prévision d'une implémentation de l'algorithme de Faggins (non implémenté).
Le système de sauvegarde précédent a été conservé pour sauvegarder les offsets, les fréquences des termes, et la liste des documents indexés. 


L'index est désormais construit selon une stratégie merge-based, qui nous permet de choisir la taille maximale de l'index temporaire en RAM.

Par la suite, nous avons décidé de faire du profiling sur l'indexation (les requêtes étant presque instantanées, il n'était pas necessaire des les optimiser).
Les résultats nous ont indiqués que le stemming et la tokenisation étaient deux axes d'amélioration. La solution retenue est de mettre en cache les termes (pour réduire le nombre d'appel au stemmer), et de remplacer le tokenizer par un simple split avec une expression régulière pour le filtre.
De plus, nous avons organisé le parsing des articles dans un pool, afin de le paralleliser. Malheuresement, étant donnée le grand nombre d'accès au cache et aux variables partagées, cela n'a pas apporté d'amélioration notable.
Temps pour contruire l'index : 
avant optimisation : ~20 min
après optimisation : ~7 min
(8*2.4GHz, 16Go ram, fichiers sur SSD)

Le système ne souffre pas de problème de mise à l'echelle : chaque document supplémentaire est indexé en temps en constant (mais necessite une reconstructionde l'index), et le nombre de document n'influence pas le temps d'execution des requêtes.

Enfin, nous avons réalisé un serveur web afin de lancer des requêtes depuis un site.

Nous avons également exploré la piste du clustering, mais n'avons pu conclure faute de temps.

## Structure 

Le système est construit autours de 4 grands blocs, à savoir l'indexer, le système de requête, le serveur web, et le système de mise en forme des données.

### Indexer

L'indexer est le bloc qui construit l'index. Sont point d'entrée est buildIndex, qui lance la construiction de l'index sur le subset selectionné (dans le code) des données. 
Pour chaque fichier, il va extraire les articles, dont le texte (et le titre) vont être ajoutés à l'index.
Pour chaque fichier, les articles peuvent être indexés en parallèle. Afin de réduire le nombre dépendances, cette fonctionnalité est desactivée.
Les mots des textes sont séparés par un split(), les symboles spéciaux sont éliminés par des expressions régulières, les mots individuels sont réduits par stemming (avec cache). 
Lorsque l'index dépasse la taille maximale, il est purgé vers une Posting List partielle.
Une fois tous les fichier explorés, les PLs partielles sont fusionnées.

### Requêtes

Le bloc de requêtes est constitué de deux système séparés de requêtes, le premier est simple, et permet d'effectuer des requêtes purement conjonctives ou disjonctive, avec tri par score.
Le second est constitué d'un lexer et d'un parser, qui permet, en se construisant autour du premier système de requètes, de conjuguer des requêtes simples avec des opérateurs pour effectuer des opérations complexe, avec des opérateurs différent selon les mots, et des parenthèses.
Ce dernier est inspiré d'un exemple de pyparsing, comme précisé par la license (inclue dans le code).
Une méthode translateTitle est fournie afin de rendre des résultats plus lisibles.

### Serveur Web

Il s'agit d'un serveur très simple, exposant une page sur le port 5366, permettant d'executer des requêtes (complexes) rapidement. 
Les résultats sont récupérés via AJAX (en utilisant jquery, hébergé sur un CDN donc requiert une connexion internet).

### Formatteur

Il s'agit d'un jeu de règles et expression réguilères permettant de transformer les documents dans un format compatible avec un parser JSON.
Les éléments non indexables (tableaux) sont retirés.

## Axes d'amélioration
Voici quelques unes de nos pistes d'amélioration :
* Indexation continue (possibilité d'ajouter des éléments à l'index)
* Utilisation des éléments autres que le titre et le texte
* Clustering
* Amélioration du serveur web
* Compression des Posting Lists
* Implémentation de Faggins (incompatible à priori avec notre système de requêtes complexes)
