# [Tansoftware](https://www.tansoftware.com) - Architecture hexagonale

![Langue : Français](https://img.shields.io/badge/langue-fran%C3%A7ais-blue)
![Licence : MIT](https://img.shields.io/badge/licence-MIT-green)
![Sujet : Architecture logicielle](https://img.shields.io/badge/sujet-architecture%20logicielle-orange)

L'**architecture hexagonale** (aussi appelée *Ports & Adapters*, c'est-à-dire « ports et
adaptateurs ») a été formalisée par Alistair Cockburn en 2005. Elle range le code d'une
application de façon à séparer ce qui constitue le cœur du métier de ce qui relève de la
technique. Le **Domain-Driven Design (DDD)**, qui veut dire « conception pilotée par le
domaine », est sa méthode compagne : il indique comment modéliser le métier que
l'architecture hexagonale se contente de ranger.

> **Que veut dire « architecture logicielle » ?** C'est la façon d'organiser les morceaux
> d'un programme entre eux : quels dossiers existent, qui a le droit d'appeler qui, où
> vivent les règles importantes. Imaginez le plan d'une maison : il décide où sont les
> murs porteurs, les couloirs et les pièces. On peut changer la peinture (la technique)
> sans toucher aux murs porteurs (le métier).

Les exemples de code sont donnés en deux langages : **Python** (lisible, peu de syntaxe
parasite, idéal pour saisir l'idée) et **PHP/Symfony** (un framework de production réel,
pour montrer la transposition concrète).

---

## Table des matières

1. [Introduction et fondamentaux](chapitres/01-introduction-et-fondamentaux.md)
   - [1. Introduction](chapitres/01-introduction-et-fondamentaux.md#1-introduction)
   - [2. Pourquoi l'hexagonal plutôt que le N-tier ?](chapitres/01-introduction-et-fondamentaux.md#2-pourquoi-lhexagonal-plutôt-que-le-n-tier-)
   - [3. Prérequis](chapitres/01-introduction-et-fondamentaux.md#3-prérequis)
   - [4. Glossaire](chapitres/01-introduction-et-fondamentaux.md#4-glossaire)
2. [Concepts architecturaux de base](chapitres/02-concepts-architecturaux-de-base.md)
   - [5. Les fondations stratégiques : DDD et hexagonal](chapitres/02-concepts-architecturaux-de-base.md#5-les-fondations-stratégiques-ddd-et-hexagonal)
   - [6. Les différentes couches](chapitres/02-concepts-architecturaux-de-base.md#6-les-différentes-couches)
   - [7. Ports et adaptateurs](chapitres/02-concepts-architecturaux-de-base.md#7-ports-et-adaptateurs)
   - [8. L'inversion de dépendance](chapitres/02-concepts-architecturaux-de-base.md#8-linversion-de-dépendance)
3. [Modélisation tactique du domaine](chapitres/03-modelisation-tactique-du-domaine.md)
   - [9. Modélisation tactique : agrégats, entités, value objects](chapitres/03-modelisation-tactique-du-domaine.md#9-modélisation-tactique-agrégats-entités-value-objects)
   - [10. Le pattern Repository](chapitres/03-modelisation-tactique-du-domaine.md#10-le-pattern-repository)
   - [11. Événements de domaine et communication asynchrone](chapitres/03-modelisation-tactique-du-domaine.md#11-événements-de-domaine-et-communication-asynchrone)
4. [Tests, comparaisons et CQRS](chapitres/04-tests-comparaisons-et-cqrs.md)
   - [12. Bénéfices en testabilité (TDD)](chapitres/04-tests-comparaisons-et-cqrs.md#12-bénéfices-en-testabilité-tdd)
   - [13. Hexagonal vs Clean Architecture vs Onion](chapitres/04-tests-comparaisons-et-cqrs.md#13-hexagonal-vs-clean-architecture-vs-onion)
   - [14. Hexagonal et CQRS : commandes, requêtes, lecture](chapitres/04-tests-comparaisons-et-cqrs.md#14-hexagonal-et-cqrs-commandes-requêtes-lecture)
5. [Câblage, mise à l'échelle et migration](chapitres/05-cablage-mise-a-lechelle-et-migration.md)
   - [15. Composition Root et câblage](chapitres/05-cablage-mise-a-lechelle-et-migration.md#15-composition-root-et-câblage)
   - [16. Hexagonal à l'échelle : monolithe modulaire et microservices](chapitres/05-cablage-mise-a-lechelle-et-migration.md#16-hexagonal-à-léchelle-monolithe-modulaire-et-microservices)
   - [17. Migrer un legacy Symfony vers l'hexagonal](chapitres/05-cablage-mise-a-lechelle-et-migration.md#17-migrer-un-legacy-symfony-vers-lhexagonal)
6. [Exemples complets](chapitres/06-exemples-complets.md)
   - [18. Exemple complet en Python](chapitres/06-exemples-complets.md#18-exemple-complet-en-python)
   - [19. Symfony en hexagonal : exemple complet](chapitres/06-exemples-complets.md#19-symfony-en-hexagonal-exemple-complet)
7. [Pièges et limites](chapitres/07-pieges-et-limites.md)
   - [20. Anti-patterns et pièges courants](chapitres/07-pieges-et-limites.md#20-anti-patterns-et-pièges-courants)
   - [21. Quand ne PAS utiliser l'hexagonal ?](chapitres/07-pieges-et-limites.md#21-quand-ne-pas-utiliser-lhexagonal-)

---

## 22. Pour aller plus loin

- Alistair Cockburn, *Hexagonal Architecture*, 2005 : l'article fondateur.
- Robert C. Martin, *Clean Architecture*, 2017 : vision élargie avec les *use cases* et
  la règle de dépendance.
- Eric Evans, *Domain-Driven Design*, 2003 : le livre bleu, fondations stratégiques et
  tactiques.
- Vaughn Vernon, *Implementing Domain-Driven Design*, 2013 : le livre rouge, exemples
  concrets et patterns d'implémentation.
- Vaughn Vernon, *Domain-Driven Design Distilled*, 2016 : une introduction synthétique
  au DDD pour qui ne veut pas avaler les 600 pages du livre rouge.
- Tom Hombergs, *Get Your Hands Dirty on Clean Architecture*, 2019 : exemple Java/Spring
  complet, transposable sans douleur en PHP/Symfony.
- Carlos Buenosvinos, Christian Soronellas et Keyvan Akbary, *Domain-Driven Design in
  PHP*, 2017 : le pendant PHP du livre rouge, avec exemples Symfony/Doctrine.
- Matthias Noback, *Object Design Style Guide* et *Advanced Web Application Architecture* :
  beaucoup d'idées concrètes pour structurer un projet PHP moderne.

[Retour en haut](#table-des-matières)

---

## 23. Auteur

**Tanguy Chénier**, Tansoftware

- Site : [tansoftware.com](https://www.tansoftware.com)
- LinkedIn : [linkedin.com/in/tanguy-chenier](https://www.linkedin.com/in/tanguy-chenier/)
- GitHub Tan-Software (organisation) : [github.com/Tan-Software](https://github.com/Tan-Software)
- GitHub personnel (derniers outils) : [github.com/tanguychenier](https://github.com/tanguychenier)

[Retour en haut](#table-des-matières)

---

## 24. Licence

Distribué sous licence **MIT** : voir le fichier [LICENCE](./LICENCE).

[Retour en haut](#table-des-matières)
