[← Exemples complets](06-exemples-complets.md) · [↑ Sommaire](../README.md#table-des-matières)

# 7. Pièges et limites

## 20. Anti-patterns et pièges courants

Chaque ligne nomme une erreur fréquente, la décrit, puis donne le réflexe correct.

| Piège | Description | Comment l'éviter |
|---|---|---|
| **Anemic domain model** | Entités sans comportement, juste des sacs de getters/setters. La logique fuite dans des `*Service` énormes. | Mettre les invariants et les transitions d'état **dans** les entités. Méthodes métier nommées (`markDone`), pas `setDone($b)`. |
| **Leaky infrastructure** | Le domaine importe `psycopg2`, `Doctrine\ORM\Mapping`, `Symfony\HttpFoundation`… | Aucune dépendance externe dans `Domain/`. Vérifier avec un test d'architecture (ex. `deptrac` en PHP, `archunit`/`import-linter`). |
| **Fat use cases** | Use cases de 500 lignes qui font tout : validation, logique métier, persistance, e-mail. | Découper par cas d'utilisation, extraire les services de domaine, déléguer la validation à un VO. |
| **Faux ports** | Une « interface » qui ne fait que recopier la classe concrète, méthode pour méthode. | Concevoir les ports depuis les besoins du domaine, pas depuis l'implémentation. Les noms de méthodes doivent être métier. |
| **Repository fourre-tout** | `findBy(array $criteria)` qui accepte tout. | Méthodes nommées par cas d'usage : `findUnpaidOrders()`, `findCustomersDueForReminder()`. |
| **Dépendance circulaire** *(domain ↔ application)* | Le domaine appelle un use case. | Le domaine ne connaît que lui-même. Les use cases orchestrent. |
| **DTO confondus avec entités** | Renvoyer une entité directement à l'API. | Convertir en DTO/ViewModel dans l'adaptateur primaire. Ne jamais sérialiser une entité. |
| **Sur-architecture** | Hexagonal pour un script de 200 lignes. | Voir section [21](#21-quand-ne-pas-utiliser-lhexagonal-). |
| **Adapter qui appelle un autre adapter** | `SmtpNotifier` qui appelle directement `PostgresUserRepository`. | Toujours passer par un use case ou par le domaine. Les adaptateurs ne se parlent pas entre eux. |
| **Domaine couplé au framework** | `extends AbstractController`, `implements MessageHandlerInterface` dans le domaine. | Le domaine est PHP pur (POPO). Les adaptateurs *seuls* étendent les classes du framework. |
| **Pas de bounded context** | Tout le code dans un seul `src/Entity/`, un seul `src/Service/`, ambiguïté sur les noms. | Découper par bounded context (`src/Catalog/`, `src/Billing/`, `src/Shipping/`). |
| **Annotation ORM dans le domaine** | `#[ORM\Entity]` directement sur l'entité métier. | Soit assumer le compromis (petit projet), soit séparer entité ORM et entité domaine via un mapper. |
| **Use case qui contient une règle métier** | `if ($order->total > 1000) ...` dans le use case. | Déplacer la règle dans `Order` ou dans un service de domaine. Le use case orchestre, point. |
| **Horloge implicite** | `new \DateTimeImmutable()` au milieu du domaine. | Injecter une `ClockInterface`. Rend le domaine déterministe et testable. |
| **Identifiants générés par la DB** | Dépendre de l'auto-incrément SQL pour avoir un id. | Générer l'id côté domaine (UUID). L'agrégat naît avec son identité. |
| **Service Locator dans un use case** | Le use case reçoit un `ContainerInterface` et fait `$container->get('xxx')`. | Injecter explicitement chaque dépendance dans le constructeur. Le use case doit *déclarer* ce dont il a besoin. |
| **Classes « Manager » fourre-tout** | `OrderManager`, `UserManager` qui font tout : CRUD, règles métier, e-mail, log. | Renommer par cas d'utilisation (`PlaceOrderUseCase`), extraire la règle métier vers `Order`. Le mot « manager » est un signal d'alerte. |
| **`Request`/`Response` Symfony dans le use case** | `__invoke(Request $request): Response` pour un use case. | Les types Symfony `Request`/`Response` restent *au contrôleur*. Le use case prend une *command* et renvoie un DTO ou `void`. |
| **Test « unitaire » qui démarre Doctrine** | Test du domaine qui boot SQLite + schéma + Doctrine pour vérifier une règle métier. | Si vous ne pouvez pas tester sans I/O, c'est que vous testez l'infrastructure, pas le domaine. Renommer en *test d'intégration*. |
| **Interface inutile** | Un port qui n'a *qu'une seule* implémentation prévisible et qui ne sert pas à tester. | Tous les ports n'ont pas besoin d'exister. Si vous n'avez pas besoin de double, gardez la classe concrète. |
| **Publication d'événements hors transaction** | `repository.save(); bus.publish();` sans garantie atomique : crash entre les deux = événement perdu ou base incohérente. | Pattern *transactional outbox* : insérer l'événement dans une table `outbox` *dans la même transaction* que l'écriture, puis un worker dédié relaie vers le bus. |
| **`#[Route]` sur un use case** | Annotation HTTP collée directement sur la classe applicative. | Les routes appartiennent aux contrôleurs. Le use case ne sait *rien* du transport. |
| **CQRS sans bénéfice** | Mettre tout le monde derrière un *bus* + *handlers* « parce que CQRS » alors qu'on n'a ni read model dénormalisé ni problème de scale. | Commencer en CQRS léger (use case = commande applicative, requête = port de lecture). Ne pas mettre un bus juste pour le rituel. |
| **Mocks qui imitent l'ORM** | Test qui mocke `EntityManagerInterface::find()`, `persist()`, `flush()` pour tester un repository. | Tester *l'adaptateur* contre une vraie DB (intégration). Mocker l'EntityManager teste votre mock, pas votre code. |

[Retour en haut](#table-des-matières)

---

## 21. Quand ne PAS utiliser l'hexagonal ?

L'architecture hexagonale **a un coût** : plus de fichiers, plus d'interfaces, plus
d'indirection. Elle n'est donc pas toujours rentable.

> **Que veut dire « indirection » ?** Le fait de passer par un intermédiaire au lieu
> d'aller droit au but (par exemple parler à un port plutôt qu'à la base directement).
> Chaque intermédiaire ajoute de la souplesse mais aussi un petit détour. Analogie :
> passer par un standard téléphonique plutôt que d'appeler la personne en direct.

Cas où elle ne se justifie pas :

- **Scripts jetables et *proofs of concept*** : la lourdeur ne se rentabilise pas. Un
  *proof of concept* (« preuve de concept ») est un prototype rapide destiné à valider une
  idée, pas à durer.
- **Applications CRUD pures** sans règle métier (un simple panneau d'administration
  Symfony EasyAdmin ou Django suffit). *CRUD* veut dire *Create, Read, Update, Delete*
  (« créer, lire, modifier, supprimer ») : les quatre opérations de base sur des données,
  sans logique par-dessus.
- **Domaine très instable**, où l'on ne sait pas encore ce qu'on construit (commencer
  simple, basculer vers l'hexagonal *quand* les frontières deviennent claires).
- **Très petites équipes** plus à l'aise avec un style classique : la cohérence prime sur
  l'élégance théorique.
- **Latences critiques** sur des chemins ultra-sollicités : la double indirection (port,
  adaptateur, mapper) ajoute des nanosecondes ; rare, mais réel en HFT et en jeu vidéo
  temps réel. *HFT* veut dire *High-Frequency Trading*, « trading à haute fréquence », où
  l'on passe des milliers d'ordres de bourse par seconde et où chaque microseconde compte.

> **Règle pragmatique.** Si vous n'avez pas de logique métier non triviale, vous n'avez
> probablement pas besoin d'hexagonal. Mais dès qu'une règle métier mérite d'être testée
> seule, l'isoler dans un domaine pur paie très vite. Et plus le projet vit longtemps,
> plus l'investissement est rentable.

[Retour en haut](#table-des-matières)

---

---

[← Exemples complets](06-exemples-complets.md) · [↑ Sommaire](../README.md#table-des-matières)
