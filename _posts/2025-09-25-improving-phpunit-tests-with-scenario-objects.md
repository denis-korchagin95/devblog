---
layout: post
title: "Improving PHPUnit Tests with Scenario Objects"
date: 2025-09-25 22:03:00 +0200
categories: [testing]
tags: [phpunit, tdd, bdd]
---

# Introduction

When working with PHPUnit tests, it’s common to end up with **long, unreadable test cases** that stretch over multiple screens. They are pretty challenging to scan, hard to maintain, and prone to mistakes when business logic changes.

In this article, I will show how to improve test readability and maintainability by introducing a **Scenario Object** — an approach inspired by BDD principles that makes tests concise and expressive.

---

# The Problem

Here is a real example of a test I had to write. Notice how much boilerplate is needed just to set up a simple filter-by-project test case:

```php
#[Test]
public function aUserCanFilterTasksByProjectName(): void
{
    $entityBuilder = $this->getEntityBuilder();

    $user = $entityBuilder->asUserBuilder()->withName('Alex')->build('user');

    $today = new DateTimeImmutable('today');

    // True scenario
    $entityBuilder->asTaskBuilder()
        ->withTitle('Buy sponges and dish soap')
        ->withStatus('open')
        ->withPriority('normal')
        ->withAssignee($user)
        ->withDueDate($today)
        ->withProject(
            $entityBuilder
                ->asProjectBuilder(
                    $entityBuilder->asWorkspaceBuilder()->withName('Home chores')->build('workspace1'),
                )
                ->withName('Kitchen Helpers')
                ->withArchived(false)
                ->build('project1'),
        )
        ->build('task1');

    // False scenario
    $entityBuilder->asTaskBuilder()
        ->withTitle('Rake leaves in the backyard')
        ->withStatus('open')
        ->withPriority('low')
        ->withAssignee($user)
        ->withDueDate($today)
        ->withProject(
            $entityBuilder
                ->asProjectBuilder(
                    $entityBuilder->asWorkspaceBuilder()->withName('Home chores')->build('workspace2'),
                )
                ->withName('Backyard Cleanup')
                ->withArchived(false)
                ->build('project2'),
        )
        ->build('task2');

    $entities = $entityBuilder->persistEntities();

    $this->actingAs($user, [UserAccessLevel::ACCESS_ASSIGNEE]);
    $response = $this->get(uri: '/api/todo/list?query=chen&page=1&perPage=50');

    // Assertions...
}
```

Even for such a simple case, the test takes up **2–3 screens** and hides the core logic behind setup noise.

---

# Idea: Scenario Object

To improve readability, I introduced a **Scenario Object**.

The idea is simple:
- Build test data incrementally (Builder pattern).
- Expose **only the details relevant to the test** while hiding unnecessary setup.
- Make the test read like a **Given–When–Then** BDD scenario.

For example:

```php
$falseScenario = SearchTaskScenario::given($entityBuilder, $user)
    ->andHasWorkspace()
    ->andHasProject(name: 'Backyard Cleanup')
    ->andHasTask();

$trueScenario = SearchTaskScenario::given($entityBuilder, $user)
    ->andHasWorkspace()
    ->andHasProject(name: 'Kitchen Helpers')
    ->andHasTask();
```

At a glance, you immediately see what matters: the project names. Everything else is handled by default properties.

---

# Implementation

The core of this approach is a dedicated class that represents a **test scenario**:

```php
final class SearchTaskScenario
{
    private ?Task $task = null;
    private ?Workspace $workspace = null;
    private ?Project $project = null;

    public function __construct(
        private readonly EntityBuilder $entityBuilder,
        private readonly User $user,
    ) {}

    public static function given(EntityBuilder $entityBuilder, User $user): self
    {
        return new self($entityBuilder, $user);
    }

    public function andHasTask(
        string $name = 'Task 1',
        string $status = 'open',
        string $priority = 'low',
        ?DateTimeImmutable $dueDate = null,
    ): self {
        if ($this->task !== null) {
            throw new LogicException('The task has already been set.');
        }

        $taskBuilder = $this->entityBuilder->asTaskBuilder()
            ->withName($name)
            ->withStatus($status)
            ->withPriority($priority);

        if ($dueDate !== null) {
            $taskBuilder->withDueDate($dueDate->setTime(0, 0));
        }

        $this->task = $taskBuilder->build();

        return $this;
    }

    public function andTaskHasAssignee(?User $assignee = null): self
    {
        $this->task()->addAssignee($assignee ?? $this->user);

        return $this;
    }

    public function andHasWorkspace(string $name = 'Workspace 1'): self
    {
        if ($this->workspace !== null) {
            throw new LogicException('The workspace has already been set.');
        }

        $this->workspace = $this->entityBuilder->asWorkspaceBuilder()
            ->withName($name)
            ->build();

        return $this;
    }

    public function andHasProject(string $name = 'Project 1', bool $isArchived = false): self
    {
        if ($this->project !== null) {
            throw new LogicException('The project has already been set.');
        }

        $this->project = $this->entityBuilder->asProjectBuilder($this->workspace())
            ->withName($name)
            ->withArchived($isArchived)
            ->build();

        return $this;
    }

    public function task(): Task { return $this->task ?? throw new LogicException('Task not set'); }
    public function workspace(): Workspace { return $this->workspace ?? throw new LogicException('Workspace not set'); }
    public function project(): Project { return $this->project ?? throw new LogicException('Project not set'); }
}
```

This class hides the boilerplate while keeping test setup **explicit and readable**.

---

# Example in Practice

With `SearchTaskScenario`, the test shrinks dramatically:

```php
#[Test]
public function aUserCanFilterTasksByProjectName(): void
{
    $entityBuilder = $this->getEntityBuilder();
    $user = $entityBuilder->asUserBuilder()->withName('Alex')->build('user');
    $today = new DateTimeImmutable('today');

    $falseScenario = SearchTaskScenario::given($entityBuilder, $user)
        ->andHasWorkspace()
        ->andHasProject(name: 'Backyard Cleanup')
        ->andHasTask(dueDate: $today)
        ->andTaskHasAssignee();

    $trueScenario = SearchTaskScenario::given($entityBuilder, $user)
        ->andHasWorkspace()
        ->andHasProject(name: 'Kitchen Helpers')
        ->andHasTask(dueDate: $today)
        ->andTaskHasAssignee();

    $entityBuilder->persistEntities();

    $this->actingAs($user, [UserAccessLevel::ACCESS_ASSIGNEE]);

    $response = $this->get(uri: '/api/todo/list?query=chen&page=1&perPage=50');

    $this->assertSearchTaskResponse($response, [$trueScenario]);
}
```

Now the test fits in one screen, and it's immediately obvious what the intent is.

---

# Extending the Approach

Because a scenario encapsulates all necessary entities, you can:

- Reuse it for **assertion helpers**.
- Build custom `assertSearchTaskResponse()` that checks payloads without repeating field-by-field expectations.
- Speed up writing and maintaining tests.

---

# Conclusion

Using **Scenario Objects** for PHPUnit tests brings clarity, reduces boilerplate, and makes test intent visible at a glance. Inspired by BDD, this approach helps keep focus on *what* is being tested rather than *how* it’s set up.

👉 Try it in your test suite — I would love to hear your feedback!

# References

- [Behavior-driven development (Wikipedia)](https://en.wikipedia.org/wiki/Behavior-driven_development)
