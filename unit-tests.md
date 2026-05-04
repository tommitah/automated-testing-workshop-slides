---
theme:
    name: dark
title: Automated testing philosophy
author: Tommi Tahvanainen
date: 2026-04-21
---

# In this workshop
Learning about *what* to test, *how* to test, *why* to test and more!

In this workshop we'll go over simple ideas of how to manage complexity and validate changes with automated testing.

<!-- pause -->
## Not in this workshop
<!-- pause -->
1. Default answers on what tools (libraries/frameworks) to use for Java
<!-- pause -->
2. How to specifically setup a technology in your repositories

<!-- pause -->
### This presentation was presented via [presenterm](https://mfontanini.github.io/presenterm/introduction.html)
*NO GEN-AI WAS USED IN GENERATING THIS PRESENTATION*

<!-- end_slide -->
## On testing
Question for the audience: Do the services have interdepency?
Ie.: Service A calls service B via http/rpc?

1. Simple utility functions end up being VERY important, and should be tested. Fortunately easy to test.
2. Contracts between services should be tested (mostly end2end, also some integration testing possibly)
3. If something breaks (a bug is introduced), write a test that covers it.
4. Boy scouting rule: always leave the repo a bit better than you found it -> in this case add a test or two.
5. Avoid testing how your function/method is implemented, instead test outcomes and behaviour.

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
}
```
<!-- end_slide -->
```typescript
// test
// ... assume we have mocked the deps
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
```typescript
export const makeDocumentUseCase = ({
  someStorage
  someNotificationSystem
  changeLog
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
    }
  }
}
```
<!-- end_slide -->
```typescript
import { type Dependencies, makeDocumentUseCase } from '@mod'

const mockDeps: Dependencies = { /*...*/ }

it('should create a document record', async () => {
  const document: Document = { id: 'foo' }
  const user: User = { id: 'bar', name: 'zoo' }
  const params = { data: document, user }

  const create = makeDocumentUseCase(mockDeps)
  await create(params)

  // test outcomes
  await mockDeps.someStorage.get({ id: document.id })
})

it('should create a notification in the event system')

it('should create a change log record')
```
