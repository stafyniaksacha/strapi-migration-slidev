---
# try also 'default' to start simple
theme: default
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# use UnoCSS (experimental)
css: unocss
---

<img src="https://digisquad.io/_nuxt/digisquad.3b8713df.svg" class="w-12 mx-auto mb-4" />

# Strapi x Pubfac

Gestion des migrations des schémas et données

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Let's go <carbon:arrow-right class="inline"/>
  </span>
</div>

<div class="abs-br m-6 flex gap-2">
  <a href="https://digisquad.io" target="_blank" alt="GitHub"
    class="text-xl icon-btn opacity-50 !border-none !hover:text-white flex items-center justify-center gap-2">
    <carbon-earth/>
    <span class="text-base">digisquad.io</span>
  </a>
  <a href="https://github.com/stafyniaksacha" target="_blank" alt="GitHub"
    class="text-xl icon-btn opacity-50 !border-none !hover:text-white flex items-center justify-center gap-2">
    <carbon-logo-github />
    <span class="text-base">@stafyniaksacha</span>
  </a>
</div>

<!--
start
```
git checkout 077f9b35f0d7b04ff48a799820ace290cd052e8f
```
-->

---

# Programme

<br>

- 📝 **Articles** - présentation d'un schéma strapi
- 🎨 **Environements** - sqlite et postgres pour le dévelopement
- 🧑‍💻 **Beekeeper** - visualisation des structures et des données
- 📎 **Strapi Store** - collection interne dédié à la configuration
- 🌱 **Strapi Bootstrap** - définition des permissions, plugin stores, data-fixtures
- 🤹 **Strapi Console** - l'outil pour explorer strapi
- 📤 **Tests** - tester c'est ne plus douter, avec jest & supertest
- ⚡ **Benchmarks** - pour optimiser il faut mesurer, avec 0x & autocannon
- 📌 **Indexes** - ajout d'index sur la collection articles
- 🛠 **Migrations** - adapter les données avant le changement d'un model

<br>
<br>


---
layout: center
image: https://images.unsplash.com/photo-1457369804613-52c61a468e7d?auto=format&fit=crop&w=1024&q=80
---

# Article Schema

> `src/api/article/content-types/article/schema.json`

```jsonc {all|10-14|16-22}
{
  "kind": "collectionType",
  "collectionName": "articles",
  "info": {
    "singularName": "article",
    "pluralName": "articles",
    "displayName": "Article",
    "description": ""
  },
  "options": {
    "draftAndPublish": false,
    // populateCreatorFields: false, // hide createdBy / updatedBy (in rest response)
    // timestamps: ['createdAt', 'updatedAt'], // rename createdAt / updatedAt
  },
  "pluginOptions": {},
  "attributes": {
    "slug": { /* ... */ },
    "version": { /* ... */ },
    "state": { /* ... */ },
    "title": { /* ... */ },
    "content": { /* ... */ }
  }
}
```


---
layout: image-right
image: https://images.unsplash.com/photo-1457369804613-52c61a468e7d?auto=format&fit=crop&w=1024&q=80
---

# Article Schema (state)

```jsonc {5-16|10}
{
  "attributes": {
    "slug": { /* ... */ },
    "version": { /* ... */ },
    "state": { 
      "type": "enumeration",
      "enum": [
        "Draft",
        "In_Review",
        "Scheduled",
        "Published"
      ],
      "required": true,
      "default": "Draft",
      "private": true,
    },
    "title": { /* ... */ },
    "content": { /* ... */ }
  }
}
```

---
layout: two-cols
---


# Article Service

> `src/api/article/services/find.js`

```js
/**
 * Find one article by slug
 */
async function bySlug(slug) {
  const [article] = await strapi
    .entityService
    .findMany(
      'api::article.article', 
      {
        filters: {
          slug,
          state: 'Published',
        },
        limit: 1,
        sort: { version: 'desc' },
      }
    )

  return article
}
```

::right::

<div v-click class="px-2">

```js
/**
 * Find all articles 
 */
async function latest(options = {limit: 10, offset: 0}) {
  const knex = strapi.db.connection

  const inner = [
    "(version, slug) in",
    "(",
      "select max(version) as version, slug",
      "from articles",
      "group by slug, state",
      "having state = 'Published'",
    ")",
  ].join(' ')

  const articles = await knex('articles')
    .limit(options.limit)
    .offset(options.offset)
    .orderBy('created_at', 'desc')
    .where(knex.raw(inner))
    .select()

  return articles
}
```

</div>

---
layout: center
---


# Article Controller

> `src/api/article/controllers/find.js`

```js
module.exports = {
  async bySlug(ctx) {
    const { slug } = ctx.params

    // call our service
    const article = await strapi.service('api::article.find').bySlug(slug)

    // uniform response
    const sanitized = await strapi.controller('api::article.article').sanitizeOutput(article, ctx)
    return strapi.controller('api::article.article').transformResponse(sanitized)
  },
  async latest(ctx) {
    // call our service
    const articles = await strapi.service('api::article.find').latest()

    // uniform response
    const sanitized = await strapi.controller('api::article.article').sanitizeOutput(articles, ctx)
    return strapi.controller('api::article.article').transformResponse(sanitized)
  }
}
```

---
layout: image-right
image: https://images.unsplash.com/photo-1553002401-c0945c2ff0b0?auto=format&fit=crop&w=1024&q=80
---

# Article Routes

> `src/api/article/routes/find.js`

```js {all|7-9}
module.exports = {
  routes: [
    {
      method: 'GET',
      path: '/articles/get/latest',
      handler: 'find.latest',
      config: {
        auth: false,
      },
    },
    {
      method: 'GET',
      path: '/articles/slug/:slug',
      handler: 'find.bySlug',
      config: {
        auth: false,
      },
    },
  ],
}
```

<!--
api created
```
git checkout 9ba055a8da73fd578eae764383be315bf2784035
```
-->

---

# Environements

|     |     |
| --- | --- |
| <kbd>development</kbd> / <kbd>sqlite</kbd>| utilisé par défaut, simple et rapide<br/>`yarn develop` |
| <kbd>development-pg</kbd>  / <kbd>postgres</kbd> | utilisé à la demande<br/>`NODE_ENV=development-pg yarn develop` |
| <kbd>test</kbd> / <kbd>sqlite</kbd>| utilisé par jest<br/>`yarn test` |

<div v-click class="flex justify-center items-center gap-4 text-4xl mt-6 p-6 border-[red] rounded border max-w-[300px] mx-auto">
 <logos-postgresql />  <fluent-equal-off-20-regular class="color-[red]" />  <logos-mysql/>
</div>


---
layout: two-cols
---

# Postgres <small>(via docker-compose)</small>

> `docker-compose.local-postgres.yml`

```yaml
version: '3'

services:
  postgresql:
    image: 'bitnami/postgresql:latest'
    ports:
      - 5432:5432
    environment:
      - POSTGRESQL_USERNAME=my_user
      - POSTGRESQL_PASSWORD=password123
      - POSTGRESQL_DATABASE=my_database
```

<v-click>

```bash
docker-compose -f docker-compose.local-postgres.yml up
```
```bash
docker-compose -f docker-compose.local-postgres.yml rm -f
```

</v-click>

::right::

# &nbsp;

<div v-click class="px-2">

> `config/env/development-pg/database.js`

```js
module.exports = ({ env }) => ({
  connection: {
    client: 'postgres',
    connection: {
      host: 'localhost',
      port: 5432,
      database: 'my_database',
      user: 'my_user',
      password: 'password123',
      timezone: 'utc',
      schema: 'public',
    },
  },
});
```

</div>

<div v-click class="px-2">

```bash
DEBUG=knex:* NODE_ENV=development-pg yarn strapi develop
```

</div>

---
layout: section
---

# Beekeeper Studio


<img src="https://www.beekeeperstudio.io/assets/d39663-d7b3b0149ba94ed3dc5aec77e28da4f5592205c740e94d8eaed947d2b87c1d34.png" class="max-w-[60%] mx-auto" />

<a href="https://www.beekeeperstudio.io/">https://www.beekeeperstudio.io/</a>


<!--
postgres query fix
```
git checkout a69de5abcecca768b29454f7ecaa18a06dac4fea
```
-->

---
layout: two-cols
---

# Strapi Store

> [`core/strapi/lib/services/core-store.js`](https://github.com/strapi/strapi/blob/main/packages/core/strapi/lib/services/core-store.js)

```ts {all|3|all}
const coreStoreModel = {
  uid: 'strapi::core-store',
  collectionName: 'strapi_core_store_settings',
  attributes: {
    key: {
      type: 'string',
    },
    value: {
      type: 'text',
    },
    type: {
      type: 'string',
    },
    environment: {
      type: 'string',
    },
    tag: {
      type: 'string',
    },
  },
};
```

::right::

# &nbsp;

<div v-click class="pt-9 px-2">

```js
// read users-permissions email (template)
const value = await strapi
  .store({
    type: "plugin",
    name: "users-permissions",
  })
  .get({
    key: "email" 
  });
```

```js
// same as
const value = await strapi.store.get({
  type: "plugin",
  name: "users-permissions",
  key: "email",
})
```

</div>

---

# Strapi Bootstrap

> `src/index.js`

```js {all|3-7|9-15|17-20}
module.exports = {
  async bootstrap() {
    // override permissions
    await setPermissions('public', {
      'plugin::users-permissions.auth': ['connect', 'callback', 'forgotPassword', 'resetPassword'],
      'plugin::users-permissions.user': ['me'],
    }, { reset: true })

    // setup users-permissions providers settings
    const upStore = strapi.store({ type: "plugin", name: "users-permissions" })
    const grant = await upStore.get({ key: "grant" })
    grant.google.enabled = true
    grant.google.key = 'GKEY'
    grant.google.secret = 'GSECRET'
    await upStore.set({ key: "grant", value: grant })
    
    // load initial data
    if (await isFirstRun()) {
      await importArticles()
    }
  }
}
```

---

# Strapi Bootstrap: isFirstRun

```js
async function isFirstRun() {
  const store = strapi.store({
    type: "app",
    name: "setup",
  })

  const initHasRun = await store.get({ key: "initHasRun" });

  // persist app_setup_initHasRun in database
  await store.set({ key: "initHasRun", value: true });

  return !initHasRun;
}
```

---

# Strapi Bootstrap: importArticles

```js
const articles = [
  // article-1
  { slug: 'article-1', version: 1, state: 'Published', title: "Article N°1", content: "# Hello world\nversion 1" },
  { slug: 'article-1', version: 2, state: 'Published', title: "Article N°1", content: "# Hello world\nversion 2" },
  { slug: 'article-1', version: 3, state: 'Published', title: "Article N°1", content: "# Hello world\nversion 3" },
  { slug: 'article-1', version: 4, state: 'Scheduled', title: "Article N°1", content: "# Hello world\nversion 4" },
  { slug: 'article-1', version: 5, state: 'Draft', title: "Article N°1", content: "# Hello world\nversion 5" },

  // ...
]

async function importArticles() {
  const promises = []

  for (const article of articles) {
    promises.push(strapi.entityService.create('api::article.article', {
      data: article
    }))
  }

  await Promise.all(promises)
}
```

---

# Strapi Bootstrap: setPermissions

```js {all|4-6|10-21|all}
async function setPermissions(type, permissions, options = { reset: false }) {
  // ...

  if (options.reset) {
    await strapi.db.connection.raw('DELETE FROM "up_permissions_role_links" WHERE "role_id" = ?', [role.id]);
  }

  // ...

  for (const controllerId in permissions) {
    // ...
    const permissionsToCreate = actionIds.map((actionId) =>
      strapi.query("plugin::users-permissions.permission").create({
        data: {
          action: `${controllerId}.${actionId}`,
          role: role.id,
        },
      })
    );
    // ...
  }
  // ...
}
```

<!--
bootstrap imports 
```
git checkout 8b900685791b787debc671c2bcd4c3cf09169004
```
-->

---
layout: section
---

# Knex.js: SQL query builder


<img src="https://knexjs.org/knex-logo.png" class="max-w-[15%] my-12 mx-auto" />

<a href="https://knexjs.org/">https://knexjs.org/</a>

---
layout: image-right
image: https://images.unsplash.com/photo-1603468620905-8de7d86b781e?auto=format&fit=crop&w=1024&q=80
---

<div class="flex flex-col gap-6 mt-28 justify-center items-center">

# Strapi Console


`yarn strapi console`

</div>

---
layout: center
---

# Strapi Start Script

> `./start-strapi.js`

```js
const Strapi = require("@strapi/strapi");

async function setup() {
  // this register strapi as global!
  Strapi();
  await strapi.start();

  return Promise.resolve(strapi);
}

setup()
  .then(async (strapi) => {
    const articles = await strapi.service('api::article.find').latest()
    console.dir(articles, { depth: null })
    
    return strapi.destroy()
  })
  .catch(console.error);
```

`node ./start-strapi.js`

---
layout: two-cols
---

# Tests

`yarn add -D jest supertest`

<v-click>

```js
const { setup, teardown, agent } = require('./helpers')

describe('articles api', () => {
  beforeAll(setup)
  afterAll(teardown)

  describe('/api/articles/get/latest', () => {
    it('should respond with 200', async () => {
      expect.assertions(1)

      const response = await agent()
        .get('/api/articles/get/latest')

      expect(response.status).toBe(200)
    })
  })
})
```

</v-click>

::right::

<div v-click class="px-2">

```js
const { existsSync, unlinkSync } = require('node:fs');
const Strapi = require('@strapi/strapi');
const supertest = require('supertest');

async function setup() { /* ... */ }

function agent() {
  return supertest.agent(strapi.server.httpServer);
}

async function teardown() {
  // close server to release the db-file
  await strapi.destroy();

  // delete test database after all tests
  const dbSettings = strapi.config.get('database.connection.connection');
  if (dbSettings && dbSettings.filename) {
    const tmpDbFile = `${dbSettings.filename}`;

    if (existsSync(tmpDbFile)) unlinkSync(tmpDbFile);
  }

  return Promise.resolve();
}

```

</div>

<!--
add tests
```
git checkout 0eb46a0ebc7720983d00ac1fcebe46791427a7e1
```
-->

---
layout: image-right
image: https://images.unsplash.com/photo-1532444458054-01a7dd3e9fca?auto=format&fit=crop&w=1024&q=80
---

<div class="flex flex-col gap-6 mt-28 justify-center items-center">

# Benchmarks

`yarn profile`

</div>

<!--
add benchmarks
```
git checkout 0b08bc2a178514509e3e89421fe21e2cc7a5fe19
```
-->

---

# Schema: indexes

> `src/api/article/content-types/article/schema.json`

```jsonc {all|7-12}
{
  "kind": "collectionType",
  "collectionName": "articles",
  "info": { /* ... */ },
  "options": { /* ... */ },
  "pluginOptions": {},
  "indexes": [
    {
      "name": "articles_slug_version_index",
      "columns": ["version", "slug"]
    }
  ],
  "attributes": { /* ... */ }
}
```

<!--
add indexes
```
git checkout 7c4d83a3192efa5311ffb37f5d464fde6d73c8bc
```
-->

---

# Schema: columnType

> `src/api/article/content-types/article/schema.json`

```jsonc {all|13-19}
{
  "kind": "collectionType",
  "collectionName": "articles",
  "info": { /* ... */ },
  "options": { /* ... */ },
  "attributes": { 
    "state": {
      "type": "enumeration",
      "enum": [ "Draft", "In_Review", "Published" ],
      "required": true,
      "default": "Draft",
      "private": true,
      "columnType": {
        "type": "enum",
        "args": [
          [ "Draft", "In_Review", "Published" ], 
          { "useNative": true, "enumName": "articles_publish_state" }
        ]
      }
    },
   }
}
```

---

# Migrations

`database/migrations/00-articles-state-remove-scheduled.js`

```js
module.exports = {
  async up(knex) {
    // You have full access to the Knex.js API with an already initialized connection to the database
    // ------------------------
    // EXAMPLE: renaming a table
    // await knex.schema.renameTable('oldName', 'newName');
    // ------------------------
    // EXAMPLE: renaming a column
    // await knex.schema.table('someTable', table => {
    //   table.renameColumn('oldName', 'newName');
    // });
    // ------------------------
    // EXAMPLE: updating data
    // await knex.from('someTable').update({ columnName: 'newValue' }).where({ columnName: 'oldValue' });
    // ------------------------
  },
  async down(knex) {
    // This one isn't implemented yet, will be eventually
  }
};
```

<v-click>

`strapi start --no-migrate`

</v-click>


<!--
end: columnType & migration
```
git show 515da3355ce08bfb49f68dc07f60605d92542d60
git checkout main
```
-->

---

# Ressources & Questions

- https://github.com/strapi/documentation/pull/889
- https://cs.github.com/strapi/strapi
- https://www.nearform.com/blog/tuning-node-js-app-performance-with-autocannon-and-0x/
- https://www.slowquerylog.com/
- https://knexjs.org/
- https://www.beekeeperstudio.io/

<div class="abs-br m-6 flex gap-2">
  <a href="https://digisquad.io" target="_blank" alt="GitHub"
    class="text-xl icon-btn opacity-50 !border-none !hover:text-white flex items-center justify-center gap-2">
    <carbon-earth/>
    <span class="text-base">digisquad.io</span>
  </a>
  <a href="https://github.com/stafyniaksacha" target="_blank" alt="GitHub"
    class="text-xl icon-btn opacity-50 !border-none !hover:text-white flex items-center justify-center gap-2">
    <carbon-logo-github />
    <span class="text-base">@stafyniaksacha</span>
  </a>
</div>

