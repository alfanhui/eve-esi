# Eve Online ESI Client

***Very much a work-in-progress!***

Node.js client for Eve Online ESI. Allows you to manage characters and tokens,
and make authenticated and unauthenticated requests.

Not published to npm yet. To use the module, clone the repository.

This module makes use of my other Eve related module, [https://github.com/MichielvdVelde/eve-sso](eve-sso).
Documentation is lacking, see the source code for guidance if you're brave enough to try it out.

## Example

This example shows how to authenticate a character and make a request.

The accompanied memory provider is meant solely for development, you should not
use it in production. It can also be used as a reference implementation.

```ts
'use strict'

import ESI from './index'
import MemoryProvider from './providers/memory'

import Koa from 'koa'
import Router from 'koa-router'

const provider = new MemoryProvider()

const esi = new ESI({
  provider,
  clientId: '<your client id>',
  secretKey: '<your secret>',
  callbackUri: '<your callback uri>'
})

const app = new Koa()
const router = new Router()

router.get('/', async ctx => {
  const redirectUrl = esi.getRedirectUrl('some-state', 'esi-skills.read_skills.v1')

  ctx.body = `<a href="${redirectUrl}">Log in using Eve Online</a>`
})

router.get('/sso', async ctx => {
  const code: string = ctx.query.code
  const { character } = await esi.register(code)

  ctx.res.statusCode = 302
  ctx.res.setHeader('Location', `/welcome/${character.characterId}`)
})

interface Skills {
  skills: [{
    skill_id: number,
    active_skill_level: number
  }],
  total_sp: number,
  unallocated_sp: number
}

router.get('/welcome/:characterId', async ctx => {
  const characterId = Number(ctx.params.characterId)
  const character = await provider.getCharacter(characterId)
  const token = await provider.getToken(characterId, 'esi-skills.read_skills.v1')

  let body = `<h1>Welcome, ${character.characterName}!</h1>`

  const skills = await esi.request<Skills>(
    `/characters/${characterId}/skills/`,
    null,
    null,
    { token }
  )

  const json = await skills.json()

  body += `<p>You have ${json.total_sp} total skill points.</p><ul>`

  for (const skill of json.skills) {
    body += `<li>${skill.skill_id}: ${skill.active_skill_level}</li>`
  }
  
  body += '</ul>'

  ctx.body = body
})

app.use(router.middleware())
app.listen(3001, () => {
  console.log('- Server listening')
})
```

## License

Copyright 2020 Michiel van der Velde.

This software is licensed under [the MIT License](LICENSE).
