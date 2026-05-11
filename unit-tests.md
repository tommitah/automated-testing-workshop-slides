---
theme:
    name: terminal-dark
title: Automated testing philosophy
author: Tommi Tahvanainen
date: 2026-04-21
---

# In this workshop
<!-- pause -->
Learning about *what* to test, *how* to test, *why* to test and more!

<!-- pause -->
In this workshop we'll go over simple ideas of how to manage complexity and validate changes with automated testing.

<!-- pause -->
## Not in this workshop
<!-- pause -->
1. Default answers on what tools (libraries/frameworks) to use for Java
<!-- pause -->
2. How to specifically setup a technology in your repositories

<!-- end_slide -->
# Caveat
![image:width:100%](./Asterisk-PNG-Transparent-3413446704.png)
<!-- end_slide -->

### This presentation was presented via [presenterm](https://mfontanini.github.io/presenterm/introduction.html)
*NO GEN-AI WAS USED IN GENERATING THIS PRESENTATION*

<!-- end_slide -->
## What is your experience with testing?
<!-- pause -->
1. Why do you not have tests?
<!-- pause -->
2. What kind of issues have you identified that could be helped with testing?
<!-- pause -->
3. How does the internal process work: what kind of changes have often broken something that was hard to catch by manual testing?
<!-- pause -->

## What are automated tests actually?
<!-- column_layout: [3, 2] -->
<!-- column: 0 -->
<!-- pause -->
- Programs that report: Does the code behave as we expect?
<!-- pause -->
- Programs that formalise: What is it we expect?
<!-- pause -->
- Programs that document: How is this used?
<!-- pause -->
- Programs that PROTECT: Did my change break something?
<!-- pause -->

<!-- column: 1 -->
>Tests by definition are about preventing unintended changes and protecting your sanity while making changes to your programs.
<!-- reset_layout-->
<!-- pause -->
## White box vs Black box
<!-- pause -->
### White box (or transparent tests)
>[Wiki](https://en.wikipedia.org/wiki/White-box_testing)

Tests "are aware" of the inner workings of the module/unit, can test specific internal states and paths.
<!-- pause -->
### Black box (or spec testing)
>[Wiki](https://en.wikipedia.org/wiki/Black-box_testing)

Tests "are not aware" of the inner workings, instead testing behaviour as if a client/user would.
<!-- end_slide -->
## Kinds of tests
<!-- pause -->
1. Unit tests: programs that test a `unit` of code in isolation (protect local behaviour and design decisions).
<!-- new_line -->
<!-- pause -->
2. Integration tests: programs that test along the multitude of `seams` in the application (protect collaboration between components).
<!-- new_line -->
<!-- pause -->
3. End2End tests: programs that test complete behaviour (protect user-visible workflows).
<!-- new_line -->
<!-- pause -->
<!-- end_slide -->
## Rules of thumb
1. Simple utility functions often end up being VERY important, and should be covered by tests. Fortunately these are usually easy.
<!-- new_line -->
<!-- pause -->
2. In the same vein, data transformations of any kind should be covered.
<!-- new_line -->
<!-- pause -->
3. Contracts between services can be tested (mostly end2end, also some integration testing possibly).
<!-- new_line -->
<!-- pause -->
4. If something breaks (a bug is introduced), write a test that covers it.
<!-- new_line -->
<!-- pause -->
5. Boy scouting rule: always leave the repo a bit better than you found it -> in this case add a test or two.
<!-- new_line -->
<!-- pause -->
6. To help with boy scouting, you can implement a coverage checker in your CI/CD that prevents you merging if you didn't improve the coverage.
<!-- new_line -->
<!-- pause -->
7. Avoid testing how your function/method is implemented, instead test outcomes and behaviour.
<!-- new_line -->
<!-- pause -->
8. Prefer testing the public api's of modules/components.
<!-- new_line -->
<!-- pause -->
9. Avoid very elaborate testing setups per file.
<!-- new_line -->
<!-- pause -->
10. Only write tests you have demonstrated (to yourself) can fail. SOME input should break your test.

<!-- end_slide -->
## Invariant
> noun. The properties of a program that must always remain true

Invariants in software can be:
- validation rules
- specifications/requirements of third party software: think 3rd party library needs to support some feature that the code using it expects.
- business logic: switching databases or validation libraries should break tests if they don't fit into your contracts.
<!-- end_slide -->
## Testing invariants
You should be testing invariants ie. the bit that diddles with data or control flow depending on the state of the universe:
```typescript
// Instead of this
export const create = async (data: Record<string, unknown>): SomeOtherData => {
  if (!data.foo) {
    throw new Error('no foo man')
  }
  if (!data.bar) {
    throw new Error('no bar man')
  }

  const result = await db.execute(`INSERT INTO cuckoo (foo, bar) VALUES (?, ?)`, data.foo, data.bar)
  return result
}

//----------------------------//

// Do this, and test against `validateData` logic if needed
export const validateData = async (data: Record<string, unknown>): boolean => {
  if (!data.foo) {
    throw new Error('no foo man')
  }
  if (!data.bar) {
    throw new Error('no bar man')
  }
}

// We probably don't need to test the create function itself in unit tests.
export const create = async (data: Record<string, unknown>): SomeOtherData => {
  const result = await db.execute(`INSERT INTO cuckoo (foo, bar) VALUES (?, ?)`, data.foo, data.bar)
  return result
}
```
<!-- end_slide -->
## Bad tests
```typescript
// entity/document.ts
// impl
import * as someStorage from '@some-storage'
import * as someNotificationSystem from '@some-notification-system'
import * as changeLog from '@change-log'
import type { User } from '@entity/auth/user'

type Document = { id: string }
export const create = async ({
  data,
  user
}: {
  data: Record<string, unknown>,
  user: User
}): Promise<Document> => {
  const document: Document = someDomainMapper(data)
  await someStorage.insert({ data: document })

  // side effects
  // notify some event system that a document was created
  await someNotificationSystem.notify(document.id)
  // write to a auditable log what was created and by whom
  await changeLog.create({ ...document, userId: user.id, createdAt: new Date() })

  return document
}
```
<!-- end_slide -->
```typescript
//// AVOID THIS
// This test just presupposes our code is correct, and mirrors its implementation. These kinds of tests are hard to break.
import { vi } from 'vitest'

vi.mock('@some-storage', () => ({ insert: vi.fn().mockResolvedValue(undefined) }))
vi.mock('@some-notification-system', () => ({ notify: vi.fn().mockResolvedValue(undefined) }))
vi.mock('@change-log', () => ({ create: vi.fn().mockResolvedValue(undefined) }))

it('should create a document record and do N side effects', async () => {
  const document: Document = { id: 'foo' }
  const user: User = { id: 'bar', name: 'zoo' }
  const params = { data: document, user }

  await create(params)

  expect(someStorage.insert).to.have.been.called()
  expect(someStorage.insert).to.have.been.calledWith(someDomainMapper(params))
  expect(someNotificationSystem.notify).to.have.been.calledWith(document.id)
  expect(changeLog.create).to.have.been.called()
})
```
<!-- end_slide -->
## Good tests
Test outcomes instead, some change in implementation is also needed:

```typescript
// entity/document.ts
// impl
import type { SomeStorage } from '@some-storage'
import type { SomeNotificationSystem } from '@some-notification-system'
import type { ChangeLog } from '@change-log'
import type { User } from '@entity/auth/user'

type Document = { id: string }
type CreateParams = {
  data: Record<string, unknown>,
  user: User
}
type Dependencies = {
  someStorage: SomeStorage,
  someNotificationSystem: SomeNotificationSystem,
  changeLog: ChangeLog
}
type UseCase = { create: (params: CreateParams) => Promise<Document> }
```
<!-- end_slide -->
```typescript
export const makeDocumentUseCase = ({
  someStorage,
  someNotificationSystem,
  changeLog,
}: Dependencies): UseCase => {
  return {
    create: async ({ data, user }: CreateParams): Promise<Document> => {
      const document: Document = someDomainMapper(data)
      await someStorage.insert({ data: document })

      // side effects
      // notify some event system that a document was created
      await someNotificationSystem.notify(document.id)
      // write to a auditable log what was created and by whom
      await changeLog.create({ ...document, userId: user.id, createdAt: new Date() })

      return document
    }
  }
}
```
<!-- end_slide -->
<!-- column_layout: [3, 2] -->
<!-- column: 0 -->
```typescript
import { writeFile, readFile } from 'fs/promises'
import { setTimeout } from 'node:timers/promises'
import { type Dependencies, type Document, makeDocumentUseCase } from '@mod'
import { CustomError } from '@custom-error'

const LOCAL_FILE_STORAGE = './local-storage.json'
// This is pseudo code
const inMemoryDeps: Dependencies = {
  // use filesystem as storage
  someStorage: {
    insert: async ({ data }: { data: Record<string, unknown> }) => {
      await writeFile(LOCAL_FILE_STORAGE, JSON.stringify(data))
      return data
    },
    get: async ({ id }: { id: string }) => {
      try {
        const file = await readFile(LOCAL_FILE_STORAGE, { encoding: 'utf8' })
        const collection = JSON.parse(file) as Document[]
        const document = collection.find((doc) => doc.id === id)
        return document
      } catch (err) {
        // for example
        throw new CustomError(404, 'No document found')
      }
    }
  },
  // ....
}
```
<!-- column: 1 -->
```typescript
const inMemoryDeps: Dependencies = {
  // ...
  someNotificationSystem: {
    notify: async (id: string) => {
      await setTimeout(1000)
      console.log('NOTIFIED')
      return id
    }
  },
  // use filesystem as storage
  changeLog: {
    create: async (data: Record<string, unknown> & { userId: string; createdAt: Date }) => {
      await writeFile('./local-log.json', JSON.stringify(data))
      return data
    }
  }
}
```
<!-- reset_layout-->
<!-- end_slide -->
```typescript
it('should create a document record', async () => {
  const document: Document = { id: 'foo' }
  const user: User = { id: 'bar', name: 'zoo' }
  const params = { data: document, user }
  const create = makeDocumentUseCase(inMemoryDeps)
  await create(params)

  // test outcomes
  await inMemoryDeps.someStorage.get({ id: document.id })
})

it('should create a notification in the event system')
it('should create a change log record')
```
<!-- end_slide -->
## DI vs Mocks

When testing integrated code with side effects, prefer injected fakes (ie. DI) vs mocking.
<!-- new_line -->
<!-- pause -->
- DI is good at providing a transparent way of "mocking" behaviour your code depends on.
-> Less managing mock state during your test suite, and less "forced" application states.
-> Improves testing outcomes instead of implementation when needed.
<!-- pause -->
- Mocking is good for replacing thin wrappers or testing against a third party library like a Queue service from your cloud provider.
<!-- pause -->
- Mocking is fine for functions/methods that are "Orchestrators" or pipeline'y in general and don't have a lot of invariants (ie. rules about state).

<!-- end_slide -->
## Was that even worth it?
The previous slides beg the question whether we're REALLY talking about unit testing at all anymore. The tests are testing complicated stuff with enough fakery and mockery to not be considered code that actually runs in production.
I'd posit, that writing these kinds of tests is often a waste of time and especially in typed languages we should be covering the data processing parts and invariants, which are commonly regarded as business logic.

Integration testing can be useful however, when dealing with complex multi-instanced applications such as microservice architecture. It requires different tooling and methods.
<!-- end_slide -->
## Unit tests
Unit tests should cover:
- Policy: guards like `isAdmin`, `isEditable` etc.
- Mappers: data transformations, especially at the edges of the application like `internal-type->external-type`.
- Validations: validation is like policy, but it lives at the edges.
- Utilities: reusable general-purpose logic, often data processing related.
## Integration tests
Anything that isn't testing the complete application, but can't be reduced to input-output without a lot of mocking/faking is an integration test.

- Often very useful when doing migrations to new versions of third party software like updating to new database versions.
- Added benefit of having tests run against ACTUAL instances of other servers or storages.
- Should cover "risky" boundaries and adapters like auth, 
<!-- end_slide -->
## Good tests are ultimately about good source code quality

Your tests will only be as good as the thing they're testing; often you'll want to refactor your source code so it is more testable.
<!-- new_line -->
<!-- pause -->
Usual code smells that make code brittle will also make it hard to test:
<!-- pause -->
- Generically useful/necessary modules are accessed directly instead of through an interface in multiple places
<!-- pause -->
- "God objects/classes"
<!-- pause -->
- Leaky abstractions
<!-- pause -->

<!-- end_slide -->
## Okay, but what about the Database layer or any other non-trivial abstraction?
Right, you might not want to apply the `in-memory database` context to testing the persistence layer itself.
If your database layer uses SQL, you have no way of knowing if your SQL statements are correct without some kind of ORM-/compilation step to verify or actually running them. Same goes for many other tech with DSL's attached like DynamoDB (although dynamo has an official in-memory alternative).

In integration testing like this you can utilise [Testcontainers](https://testcontainers.com/) to spin up a docker instance of your actual database (or cache etc.) configuration and run testing against that.

This adds quite a bit of complexity, so if you're running on thin wrappers around third party software the juice might not be worth the squeeze.
<!-- end_slide -->
### Less is more
Only cover the main uses and api.
<!-- pause -->

### Writing tests will slow your development down (in the short term at least)
The velocity change can show up as delivering mergeable code a bit slower because you're writing tests.
It can also show up in changes taking longer, if brittle tests cause every change to break tests (again, less is more).

> Maybe in the age of Gen AI the question is more: how efficiently can we generate test coverage?

<!-- end_slide -->
## Elephant in the room
Gen AI has changed how we deliver software and will probably continue to do so. My findings from using AI to generate tests:
<!-- pause -->
- Static instructions to the LLM are necessary: LLM's will generate verbose testing scenarios that are very elaborate if you don't direct it to do it a certain way.
<!-- pause -->
- Test cases should probably come from the developer themselves, you can always ask the model to append your suite if you feel it's not enough. Stay critical.
<!-- pause -->
- I'm not sold on having an agent code and write tests to said code (even in different context windows) so YMMV.

<!-- end_slide -->
## Links
Resources:
- [Sairyss/backend-best-practices](https://github.com/Sairyss/backend-best-practices#testing)
- [Don't use in memory databases](https://phauer.com/2017/dont-use-in-memory-databases-tests-h2/)

Some (non-platform-specific) tech people have found useful:
- [Testcontainers](https://testcontainers.com/) For testing against "real live" containers of your dependencies like caches, db's etc.
- [Storybook](https://storybook.js.org/) OR [Ladle](https://ladle.dev/) For creating testing scenarios for your UI components and using them as a driver for development.
- [k6](https://k6.io/) For load testing your application end2end with N amounts of virtual users.
