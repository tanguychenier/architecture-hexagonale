[← Câblage, mise à l'échelle et migration](05-cablage-mise-a-lechelle-et-migration.md) · [↑ Sommaire](../README.md#table-des-matières) · [Pièges et limites →](07-pieges-et-limites.md)

# 6. Exemples complets

## 18. Exemple complet en Python

Un mini-domaine de **gestion de tâches** met en évidence les trois couches. Le code est
autonome et s'exécute tel quel. La version PHP/Symfony équivalente est donnée
[à la section 19](#19-symfony-en-hexagonal--exemple-complet).

### 18.1. Domaine

```python
# domain/task.py
from dataclasses import dataclass, field
from datetime import date
from enum import Enum
from uuid import UUID, uuid4


class Priority(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"


@dataclass
class Task:
    title: str
    due_date: date
    priority: Priority = Priority.MEDIUM
    done: bool = False
    id: UUID = field(default_factory=uuid4)

    def mark_done(self) -> None:
        if self.done:
            raise ValueError("Tâche déjà terminée")
        self.done = True

    def is_overdue(self, today: date) -> bool:
        return not self.done and today > self.due_date
```

### 18.2. Ports (application)

```python
# application/ports.py
from abc import ABC, abstractmethod
from typing import Iterable
from uuid import UUID

from domain.task import Task


class TaskRepository(ABC):  # Port secondaire
    @abstractmethod
    def add(self, task: Task) -> None: ...

    @abstractmethod
    def get(self, task_id: UUID) -> Task: ...

    @abstractmethod
    def list_all(self) -> Iterable[Task]: ...
```

### 18.3. Use case (application)

```python
# application/use_cases.py
from dataclasses import dataclass
from datetime import date
from uuid import UUID

from application.ports import TaskRepository
from domain.task import Priority, Task


@dataclass
class CreateTaskUseCase:  # Port primaire
    repo: TaskRepository

    def execute(self, title: str, due_date: date, priority: Priority) -> UUID:
        task = Task(title=title, due_date=due_date, priority=priority)
        self.repo.add(task)
        return task.id


@dataclass
class CompleteTaskUseCase:
    repo: TaskRepository

    def execute(self, task_id: UUID) -> None:
        task = self.repo.get(task_id)
        task.mark_done()
        self.repo.add(task)  # idempotent : même id
```

### 18.4. Adaptateur secondaire (infrastructure)

```python
# infrastructure/in_memory_repository.py
from typing import Dict
from uuid import UUID

from application.ports import TaskRepository
from domain.task import Task


class InMemoryTaskRepository(TaskRepository):
    def __init__(self) -> None:
        self._store: Dict[UUID, Task] = {}

    def add(self, task: Task) -> None:
        self._store[task.id] = task

    def get(self, task_id: UUID) -> Task:
        if task_id not in self._store:
            raise KeyError(f"Tâche introuvable : {task_id}")
        return self._store[task_id]

    def list_all(self):
        return list(self._store.values())
```

### 18.5. Adaptateur primaire et composition (infrastructure)

```python
# infrastructure/cli.py
from datetime import date

from application.use_cases import CompleteTaskUseCase, CreateTaskUseCase
from domain.task import Priority
from infrastructure.in_memory_repository import InMemoryTaskRepository


def main() -> None:
    repo = InMemoryTaskRepository()
    create = CreateTaskUseCase(repo)
    complete = CompleteTaskUseCase(repo)

    task_id = create.execute(
        title="Rédiger le mémo hexagonal",
        due_date=date(2026, 12, 31),
        priority=Priority.HIGH,
    )
    complete.execute(task_id)

    for task in repo.list_all():
        print(f"{task.title} -> done={task.done}")


if __name__ == "__main__":
    main()
```

### 18.6. Test unitaire pur

```python
# tests/test_task.py
from datetime import date

import pytest

from domain.task import Priority, Task


def test_mark_done_passes_a_task_to_done():
    task = Task(title="t", due_date=date(2026, 1, 1), priority=Priority.LOW)
    task.mark_done()
    assert task.done is True


def test_mark_done_twice_raises():
    task = Task(title="t", due_date=date(2026, 1, 1))
    task.mark_done()
    with pytest.raises(ValueError):
        task.mark_done()


def test_is_overdue():
    task = Task(title="t", due_date=date(2026, 1, 1))
    assert task.is_overdue(date(2026, 6, 1)) is True
    assert task.is_overdue(date(2025, 12, 1)) is False
```

Aucun de ces tests n'a besoin d'une base de données, d'un serveur HTTP ou d'un framework :
c'est l'objectif principal de l'architecture hexagonale.

[Retour en haut](#table-des-matières)

---

## 19. Symfony en hexagonal : exemple complet

Symfony est un framework PHP qui se prête bien à une organisation hexagonale, à condition de
**résister à la tentation** de tout coller dans des `Bundle`/`Service`/`Controller` collés à
Doctrine. Le même mini-domaine de gestion de tâches est décliné ici en PHP 8.2+ / Symfony 7.

> **Que veut dire « framework » ?** Un cadre de travail prêt à l'emploi : un ensemble
> d'outils et de conventions qui fait déjà le gros du travail répétitif (routage,
> sécurité, accès base). Symfony et Laravel sont des frameworks PHP. Analogie : une cuisine
> équipée louée, où le four et l'évier sont déjà installés ; vous apportez vos recettes.

### 19.1. Mapping conceptuel

> **Que veut dire « mapping conceptuel » ?** Une table de correspondance qui dit : « tel
> élément de Symfony joue le rôle de telle pièce hexagonale ». *Mapping* veut dire « mise
> en correspondance ». C'est un plan de traduction entre deux vocabulaires.

| Élément Symfony | Couche hexagonale | Rôle |
|---|---|---|
| `Controller` | Adaptateur primaire (HTTP) | Reçoit la requête, valide, appelle un use case, sérialise la réponse |
| `Console\Command` | Adaptateur primaire (CLI) | Lance un use case depuis la ligne de commande |
| `Messenger Handler` | Adaptateur primaire (message) | Réagit à un message asynchrone en appelant un use case |
| Classe Doctrine *qui n'étend pas* `EntityRepository` | Adaptateur secondaire (persistance) | Implémente un port de domaine `XxxRepositoryInterface` |
| `Symfony\Mailer` | Adaptateur secondaire (e-mail) | Implémente un port `EmailNotifierInterface` |
| `services.yaml` | Composition / câblage | Lie chaque interface du domaine à son adaptateur |
| Use case (classe `*UseCase`) | Couche application | Orchestre le domaine |
| Entité POPO | Couche domaine | Règles et invariants métier |
| Mapper Doctrine ↔ POPO | Infrastructure | Traduit la POPO en entité ORM et inversement |

> **Que veut dire « conteneur de services » (Symfony) ?** Un composant qui *fabrique* tous
> les objets de l'application et leur fournit automatiquement leurs dépendances. Il lit la
> configuration, résout qui a besoin de qui, et crée chaque service au moment voulu.
> Analogie : un majordome qui connaît tout l'inventaire de la maison et apporte à chacun
> exactement ce qu'il demande, sans qu'on aille le chercher soi-même. (*Service* désigne ici
> simplement un objet géré par ce conteneur.)

> **Que veut dire « autowiring » (câblage automatique) ?** À partir des *types* écrits dans
> le constructeur d'une classe, le conteneur retrouve tout seul le bon service à injecter.
> Quand le type est une interface (un port), il faut lui indiquer *quelle* implémentation
> choisir au moyen d'un **alias** dans `services.yaml`. Analogie : un branchement automatique
> qui sait reconnaître la forme de chaque prise. *Compiler pass* (« passe de compilation »)
> désigne un branchement plus avancé, fait en PHP au démarrage.

### 19.2. Arborescence proposée

```
src/
  TaskManagement/                    # Bounded context "Gestion de tâches"
    Domain/
      Model/
        Task.php
        TaskId.php
        Priority.php
      Event/
        TaskCompleted.php
      Repository/
        TaskRepositoryInterface.php
    Application/
      UseCase/
        CreateTask/
          CreateTaskCommand.php
          CreateTaskUseCase.php
        CompleteTask/
          CompleteTaskCommand.php
          CompleteTaskUseCase.php
      Port/
        ClockInterface.php
    Infrastructure/
      Persistence/
        Doctrine/
          DoctrineTaskRepository.php
          TaskMapper.php
          TaskOrmEntity.php       # entité Doctrine, séparée de la POPO du domaine
      Clock/
        SystemClock.php
    UserInterface/
      Http/
        TaskController.php
      Cli/
        CreateTaskCommand.php     # adaptateur Console
```

Chaque bounded context a son propre quadruplet `Domain / Application / Infrastructure /
UserInterface`. Cette arborescence rend la frontière hexagonale **visible à l'œil nu**.

### 19.3. Le domaine en POPO

> **Que veut dire « POPO » ?** *Plain Old PHP Object*, « simple objet PHP ordinaire ». Une
> classe PHP qui n'hérite d'aucune classe de framework et ne porte aucune annotation
> technique imposée : juste du PHP pur. C'est exactement ce qu'on veut dans le domaine.
> L'équivalent en Java se nomme POJO (*Plain Old Java Object*). Analogie : un ingrédient
> brut, non transformé, qui n'appartient à aucune marque.

```php
<?php
// src/TaskManagement/Domain/Model/TaskId.php
declare(strict_types=1);

namespace App\TaskManagement\Domain\Model;

use Symfony\Component\Uid\Uuid;

final class TaskId
{
    public function __construct(public readonly string $value)
    {
        if (!Uuid::isValid($value)) {
            throw new \InvalidArgumentException('TaskId must be a valid UUID');
        }
    }

    public static function generate(): self
    {
        return new self((string) Uuid::v4());
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }
}
```

> **Tolérance pratique.** Le composant `symfony/uid` est une bibliothèque utilitaire,
> pas un framework. La règle « zéro Symfony dans le domaine » s'entend pour les
> *frameworks* (HTTP, ORM, DI). Importer un VO d'UUID est admissible si l'équipe en
> assume le précédent.

```php
<?php
// src/TaskManagement/Domain/Model/Priority.php
declare(strict_types=1);

namespace App\TaskManagement\Domain\Model;

enum Priority: string
{
    case Low = 'low';
    case Medium = 'medium';
    case High = 'high';
}
```

```php
<?php
// src/TaskManagement/Domain/Model/Task.php
declare(strict_types=1);

namespace App\TaskManagement\Domain\Model;

use App\TaskManagement\Domain\Event\TaskCompleted;

final class Task
{
    /** @var object[] */
    private array $recordedEvents = [];

    public function __construct(
        public readonly TaskId $id,
        private string $title,
        private \DateTimeImmutable $dueDate,
        private Priority $priority,
        private bool $done = false,
    ) {
        if ($title === '') {
            throw new \DomainException('Title cannot be empty');
        }
    }

    public static function create(
        string $title,
        \DateTimeImmutable $dueDate,
        Priority $priority,
    ): self {
        return new self(TaskId::generate(), $title, $dueDate, $priority);
    }

    public function markDone(\DateTimeImmutable $now): void
    {
        if ($this->done) {
            throw new \DomainException('Task already completed');
        }
        $this->done = true;
        $this->recordedEvents[] = new TaskCompleted($this->id, $now);
    }

    public function isOverdue(\DateTimeImmutable $today): bool
    {
        return !$this->done && $today > $this->dueDate;
    }

    /** @return object[] */
    public function pullDomainEvents(): array
    {
        $events = $this->recordedEvents;
        $this->recordedEvents = [];
        return $events;
    }

    // Getters lecture seule pour la couche infrastructure
    public function title(): string { return $this->title; }
    public function dueDate(): \DateTimeImmutable { return $this->dueDate; }
    public function priority(): Priority { return $this->priority; }
    public function isDone(): bool { return $this->done; }
}
```

Notez :

- aucune annotation Doctrine, aucune dépendance HTTP ;
- les invariants sont **dans** le constructeur et dans `markDone` ;
- les événements de domaine sont *enregistrés* mais non *publiés* ; c'est le use case
  qui se charge de les transmettre au bus.

### 19.4. Les ports applicatifs

```php
<?php
// src/TaskManagement/Domain/Repository/TaskRepositoryInterface.php
declare(strict_types=1);

namespace App\TaskManagement\Domain\Repository;

use App\TaskManagement\Domain\Model\Task;
use App\TaskManagement\Domain\Model\TaskId;

interface TaskRepositoryInterface
{
    public function save(Task $task): void;

    public function get(TaskId $id): Task;

    /** @return iterable<Task> */
    public function listAll(): iterable;
}
```

```php
<?php
// src/TaskManagement/Application/Port/ClockInterface.php
declare(strict_types=1);

namespace App\TaskManagement\Application\Port;

interface ClockInterface
{
    public function now(): \DateTimeImmutable;
}
```

> **Pourquoi un port `Clock` ?** Pour que `markDone` reçoive *l'heure* depuis l'extérieur
> au lieu d'appeler `new \DateTimeImmutable()` lui-même. Cela rend le domaine
> *déterministe* et testable (on injecte une horloge fixe dans les tests).

### 19.5. Le use case applicatif

```php
<?php
// src/TaskManagement/Application/UseCase/CompleteTask/CompleteTaskCommand.php
declare(strict_types=1);

namespace App\TaskManagement\Application\UseCase\CompleteTask;

final readonly class CompleteTaskCommand
{
    public function __construct(public string $taskId) {}
}
```

```php
<?php
// src/TaskManagement/Application/UseCase/CompleteTask/CompleteTaskUseCase.php
declare(strict_types=1);

namespace App\TaskManagement\Application\UseCase\CompleteTask;

use App\TaskManagement\Application\Port\ClockInterface;
use App\TaskManagement\Domain\Model\TaskId;
use App\TaskManagement\Domain\Repository\TaskRepositoryInterface;
use Symfony\Component\Messenger\MessageBusInterface;

final class CompleteTaskUseCase
{
    public function __construct(
        private TaskRepositoryInterface $tasks,
        private ClockInterface $clock,
        private MessageBusInterface $eventBus,
    ) {}

    public function __invoke(CompleteTaskCommand $cmd): void
    {
        $task = $this->tasks->get(new TaskId($cmd->taskId));
        $task->markDone($this->clock->now());
        $this->tasks->save($task);

        foreach ($task->pullDomainEvents() as $event) {
            $this->eventBus->dispatch($event);
        }
    }
}
```

> **Note.** `MessageBusInterface` est ici importé directement de Symfony pour la lisibilité.
> Une version plus pure définirait un port `EventPublisherInterface` dans
> `Application/Port/` et un adaptateur `MessengerEventPublisher` dans `Infrastructure/`.
> C'est un arbitrage : pureté maximale (port dédié) vs pragmatisme (Symfony Messenger
> est déjà un bus stable et bien isolé). Choisissez selon le ratio coût/bénéfice de votre
> contexte.

### 19.6. L'adaptateur secondaire Doctrine

Pour respecter la règle « domaine sans annotation ORM », on sépare l'**entité du domaine**
(`Task`, POPO) de l'**entité Doctrine** (`TaskOrmEntity`, annotée pour l'ORM). Un
*mapper* fait la traduction.

```php
<?php
// src/TaskManagement/Infrastructure/Persistence/Doctrine/TaskOrmEntity.php
declare(strict_types=1);

namespace App\TaskManagement\Infrastructure\Persistence\Doctrine;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ORM\Table(name: 'task')]
class TaskOrmEntity
{
    #[ORM\Id]
    #[ORM\Column(type: 'uuid')]
    public string $id;

    #[ORM\Column(type: 'string', length: 255)]
    public string $title;

    #[ORM\Column(type: 'datetime_immutable')]
    public \DateTimeImmutable $dueDate;

    #[ORM\Column(type: 'string', length: 16)]
    public string $priority;

    #[ORM\Column(type: 'boolean')]
    public bool $done;
}
```

```php
<?php
// src/TaskManagement/Infrastructure/Persistence/Doctrine/TaskMapper.php
declare(strict_types=1);

namespace App\TaskManagement\Infrastructure\Persistence\Doctrine;

use App\TaskManagement\Domain\Model\Priority;
use App\TaskManagement\Domain\Model\Task;
use App\TaskManagement\Domain\Model\TaskId;

final class TaskMapper
{
    public function toOrm(Task $task): TaskOrmEntity
    {
        $orm = new TaskOrmEntity();
        $orm->id = $task->id->value;
        $orm->title = $task->title();
        $orm->dueDate = $task->dueDate();
        $orm->priority = $task->priority()->value;
        $orm->done = $task->isDone();
        return $orm;
    }

    public function toDomain(TaskOrmEntity $orm): Task
    {
        return new Task(
            new TaskId($orm->id),
            $orm->title,
            $orm->dueDate,
            Priority::from($orm->priority),
            $orm->done,
        );
    }
}
```

```php
<?php
// src/TaskManagement/Infrastructure/Persistence/Doctrine/DoctrineTaskRepository.php
declare(strict_types=1);

namespace App\TaskManagement\Infrastructure\Persistence\Doctrine;

use App\TaskManagement\Domain\Model\Task;
use App\TaskManagement\Domain\Model\TaskId;
use App\TaskManagement\Domain\Repository\TaskRepositoryInterface;
use Doctrine\ORM\EntityManagerInterface;

final class DoctrineTaskRepository implements TaskRepositoryInterface
{
    public function __construct(
        private EntityManagerInterface $em,
        private TaskMapper $mapper,
    ) {}

    public function save(Task $task): void
    {
        // Doctrine ORM 3 a supprimé EntityManager::merge(). On gère explicitement
        // l'insertion (entité non managée) et la mise à jour (entité existante en DB).
        $existing = $this->em->find(TaskOrmEntity::class, $task->id->value);
        $orm = $this->mapper->toOrm($task);

        if ($existing === null) {
            $this->em->persist($orm);
        } else {
            $existing->title = $orm->title;
            $existing->dueDate = $orm->dueDate;
            $existing->priority = $orm->priority;
            $existing->done = $orm->done;
        }
        $this->em->flush();
    }

    public function get(TaskId $id): Task
    {
        $orm = $this->em->find(TaskOrmEntity::class, $id->value);
        if ($orm === null) {
            throw new \DomainException("Task not found: {$id->value}");
        }
        return $this->mapper->toDomain($orm);
    }

    public function listAll(): iterable
    {
        foreach ($this->em->getRepository(TaskOrmEntity::class)->findAll() as $orm) {
            yield $this->mapper->toDomain($orm);
        }
    }
}
```

> **Mise au point Doctrine 3.** `EntityManager::merge()`, encore largement présent dans
> les tutoriels en ligne, a été *supprimé* en Doctrine ORM 3 (annoncé déprécié dès la
> 2.7). Toute documentation hexagonale qui montre encore `$em->merge($orm)` est obsolète :
> il faut `find` + `persist` (création) ou mutation des champs de l'entité managée
> (mise à jour). Cette nuance est *exactement* le genre de détail d'infrastructure qui
> *ne doit pas* fuiter dans le port `TaskRepositoryInterface` ; celui-ci se contente de
> dire `save(Task $task): void`.

Ce découplage a un coût (deux classes au lieu d'une). Pour un petit projet, on peut
**partir** d'une entité Doctrine annotée qui *est* aussi l'entité du domaine, et
introduire le mapper le jour où l'on veut nettoyer la frontière. L'important est d'avoir
*au moins* une interface `TaskRepositoryInterface` dans le domaine.

### 19.7. L'adaptateur primaire HTTP

```php
<?php
// src/TaskManagement/UserInterface/Http/TaskController.php
declare(strict_types=1);

namespace App\TaskManagement\UserInterface\Http;

use App\TaskManagement\Application\UseCase\CompleteTask\CompleteTaskCommand;
use App\TaskManagement\Application\UseCase\CompleteTask\CompleteTaskUseCase;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class TaskController
{
    public function __construct(
        private CompleteTaskUseCase $completeTask,
    ) {}

    #[Route('/tasks/{id}/complete', methods: ['POST'])]
    public function complete(string $id): Response
    {
        try {
            ($this->completeTask)(new CompleteTaskCommand($id));
            return new JsonResponse(null, Response::HTTP_NO_CONTENT);
        } catch (\DomainException $e) {
            return new JsonResponse(['error' => $e->getMessage()], Response::HTTP_CONFLICT);
        }
    }
}
```

Le contrôleur ne fait *que* trois choses : extraire les données de la requête, instancier
une *command* applicative, et traduire la réponse (ou l'exception métier) en HTTP. Toute
règle métier est interdite ici.

### 19.8. Le conteneur de services Symfony

C'est la pièce qui *câble* l'hexagone. Le fichier `config/services.yaml` lie chaque
**interface du domaine** à son **adaptateur d'infrastructure** :

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        exclude:
            - '../src/Kernel.php'

    # Liaison interface <-> implémentation
    App\TaskManagement\Domain\Repository\TaskRepositoryInterface:
        alias: App\TaskManagement\Infrastructure\Persistence\Doctrine\DoctrineTaskRepository

    App\TaskManagement\Application\Port\ClockInterface:
        alias: App\TaskManagement\Infrastructure\Clock\SystemClock
```

Avec ces alias, l'autowiring de Symfony sait que `TaskRepositoryInterface` doit être
satisfait par `DoctrineTaskRepository` en production. Pour les tests fonctionnels, on
peut *redéfinir* l'alias vers une implémentation `InMemoryTaskRepository` dans
`config/services_test.yaml` :

```yaml
# config/services_test.yaml
services:
    App\TaskManagement\Domain\Repository\TaskRepositoryInterface:
        alias: App\TaskManagement\Tests\Fixtures\InMemoryTaskRepository
        public: true
```

C'est *exactement* la promesse de l'hexagonal : *changer un adaptateur sans toucher au
domaine ni à l'application*. Le seul fichier modifié est le câblage.

### 19.9. Tests

```php
<?php
// tests/TaskManagement/Domain/TaskTest.php
declare(strict_types=1);

namespace App\Tests\TaskManagement\Domain;

use App\TaskManagement\Domain\Model\Priority;
use App\TaskManagement\Domain\Model\Task;
use App\TaskManagement\Domain\Model\TaskId;
use PHPUnit\Framework\TestCase;

final class TaskTest extends TestCase
{
    public function testMarkDoneTransitionsToDone(): void
    {
        $task = new Task(
            TaskId::generate(),
            'Rédiger le mémo',
            new \DateTimeImmutable('2026-12-31'),
            Priority::High,
        );

        $task->markDone(new \DateTimeImmutable('2026-05-02'));

        self::assertTrue($task->isDone());
        self::assertCount(1, $task->pullDomainEvents());
    }

    public function testMarkDoneTwiceThrows(): void
    {
        $task = new Task(
            TaskId::generate(),
            't',
            new \DateTimeImmutable('2026-12-31'),
            Priority::Low,
        );
        $task->markDone(new \DateTimeImmutable('2026-05-02'));

        $this->expectException(\DomainException::class);
        $task->markDone(new \DateTimeImmutable('2026-05-03'));
    }
}
```

Aucun démarrage de Symfony, aucune base de données, aucun `KernelTestCase`. Ces tests
s'exécutent en quelques millisecondes et n'ont **aucune raison** de casser tant que la
règle métier ne change pas.

> **Que veut dire « bootstrap » ?** La phase de démarrage d'une application, où elle charge
> sa configuration et construit ses objets avant de pouvoir travailler. L'image vient de
> l'expression anglaise « se hisser par ses propres lacets ». Un test de domaine pur évite
> tout ce démarrage, d'où sa rapidité.

[Retour en haut](#table-des-matières)

---

---

[← Câblage, mise à l'échelle et migration](05-cablage-mise-a-lechelle-et-migration.md) · [↑ Sommaire](../README.md#table-des-matières) · [Pièges et limites →](07-pieges-et-limites.md)
