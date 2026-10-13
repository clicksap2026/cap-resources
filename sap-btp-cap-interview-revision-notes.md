# SAP BTP CAP Interview Revision Notes

## One-minute summary

If you remember only one page, remember these twelve facts. Each one is expanded in the sections below.

1. **CAP** (Cloud Application Programming Model) is SAP's framework for building business apps and services on SAP BTP, in Node.js or Java.
2. You describe data and services in **CDS** (Core Data Services); CAP generates the database tables and the OData API for you.
3. Project layout: `db/` for the data model, `srv/` for services and logic, `app/` for UIs, `package.json` for configuration.
4. `cds watch` runs the app locally with an in-memory SQLite database and reloads on every change.
5. **Aspects** like `cuid` (UUID key) and `managed` (createdAt, createdBy, modifiedAt, modifiedBy) add common fields in one word.
6. **Association** = a reference to another entity ("points to"); **Composition** = a part that belongs to its parent ("owns", deep insert, cascade delete).
7. A **service** exposes **projections** on entities; CAP serves it as **OData V4** by default (REST and GraphQL are also possible).
8. Custom logic goes in **event handlers**: `before` (validate), `on` (replace the standard work), `after` (enrich the result).
9. **Actions** change data (POST); **functions** only read (GET); both can be **bound** to one record or **unbound**.
10. Security is declarative: `@requires` for authentication, `@restrict` for role-based and instance-based authorization; XSUAA or IAS in production.
11. **Fiori Elements** apps are generated from annotations like `@UI.LineItem`; `@odata.draft.enabled` adds safe editing.
12. Production: **SAP HANA Cloud** via HDI containers, packaged as an **MTA** and deployed with `mbt build` and `cf deploy`; remote systems like S/4HANA are reached with `cds import` and **destinations**.

## SAP BTP basics

SAP BTP (Business Technology Platform) is SAP's cloud platform where CAP apps run, next to databases, integration and AI services.

| Term | One-line meaning |
| --- | --- |
| Global account | Your company's contract with SAP BTP; the top level. |
| Subaccount | A space inside the global account for one region and purpose, like dev or prod. |
| Cloud Foundry (CF) | The runtime most CAP apps run on; you push apps and bind services. |
| CF org and space | Inside a subaccount: the org is the CF area, spaces separate dev, test and prod. |
| Kyma | SAP's managed Kubernetes runtime; the alternative to Cloud Foundry for containers. |
| Service instance | One created copy of a BTP service, like one HANA database or one XSUAA. |
| Service binding / key | The credentials that connect an app to a service instance. |
| Entitlements | Which services and how much quota a subaccount may use. |
| SAP Business Application Studio (BAS) | SAP's browser-based IDE with CAP and Fiori tools pre-installed. |
| Trial vs free tier | Trial is free but time-limited, with HANA Cloud stopped nightly; free tier lives in a paid account. |

**Say in an interview:** "CAP apps on BTP usually run on Cloud Foundry, use HANA Cloud for data, XSUAA or IAS for login, the Destination service to reach other systems, and are developed in Business Application Studio or VS Code."

## CAP fundamentals

CAP is a framework of tools, libraries and best practices that lets you focus on the business domain while CAP handles the generic work: database access, OData, draft, security and messaging.

**Why CAP?**

- **Less code:** CRUD, filtering, paging and draft work out of the box.
- **Domain-driven:** you model the business in CDS, and the code follows the model.
- **Same code local and in the cloud:** SQLite and mocked users locally, HANA Cloud and XSUAA in production, switched by configuration.
- **Clean extension:** fits the SAP "keep the core clean" strategy for side-by-side S/4HANA extensions.

**Project layout**

| Folder or file | Contains |
| --- | --- |
| `db/` | Data model (`schema.cds`) and sample data (`db/data/*.csv`) |
| `srv/` | Service definitions (`*.cds`) and handlers (`*.js` with the same file name) |
| `app/` | UI apps and UI annotations |
| `package.json` | Dependencies and the `cds` configuration (`cds.requires`) |
| `.cdsrc.json` / `.cdsrc-private.json` | More configuration; the private one is for secrets and stays out of git |
| `mta.yaml` | Deployment descriptor for Cloud Foundry |

**Essential cds commands**

| Command | Does |
| --- | --- |
| `cds init <name>` | Creates a new project |
| `cds watch` | Runs locally with SQLite in memory and auto-restart |
| `cds add <feature>` | Adds hana, xsuaa, mta, approuter, sample data, tests, and more |
| `cds compile <files> --to sql` or `--to edmx` | Shows the generated SQL or OData metadata |
| `cds deploy --to sqlite` | Creates a persistent SQLite file |
| `cds import <file>` | Imports an external API (EDMX, OpenAPI) as a remote service |
| `cds env` | Shows the effective configuration |
| `cds build` | Prepares the deployable artifacts in `gen/` |

**Profiles:** configuration can differ by profile, written as `"[production]": {...}` in package.json. `cds watch` uses `development`; deployed apps use `production`; you can add your own, like `--profile sandbox`.

**Node.js vs Java:** both are fully supported. Node.js is quicker to start and most common in tutorials; Java (Spring Boot) suits teams with Java skills and some large enterprise setups.

## CDS data modeling

CDS (Core Data Services) is the language for describing data and services; CAP compiles it to SQL tables, OData metadata and more.

```cds
namespace sales;
using { cuid, managed } from '@sap/cds/common';

entity Orders : cuid, managed {
  orderNo  : String(20) @mandatory;
  status   : String enum { New; Shipped; Cancelled } default 'New';
  customer : Association to Customers;                         // to-one
  items    : Composition of many OrderItems on items.order = $self; // owns its items
}
```

| Concept | Remember |
| --- | --- |
| Entity | A table-like business object; becomes a database table. |
| Namespace | A prefix for all names, like `sales.Orders`; avoids clashes. |
| Key | `key ID : UUID`, or use the `cuid` aspect. |
| Built-in types | String, Integer, Decimal(p,s), Boolean, Date, DateTime, Timestamp, UUID, LargeString. |
| Custom type | `type Amount : Decimal(10,2);` reused across entities. |
| Enum | A fixed list of allowed values, like status codes. |
| Aspect | A reusable set of fields: `cuid`, `managed`, `temporal`, or your own `aspect`. |
| `managed` | Fills createdAt, createdBy, modifiedAt, modifiedBy automatically. |
| Association | A reference; managed (`Association to X`, CAP adds the foreign key `x_ID`) or unmanaged (with an `on` condition). |
| Composition | Parent owns children: deep insert, deep update and cascade delete come for free. |
| Backlink | To-many association: `Association to many Orders on orders.customer = $self`. |
| `$self` | Refers to the current entity in an `on` condition. |
| Calculated element | `total : Decimal = price * quantity;` calculated on read, not stored. |
| Virtual element | `virtual seatsLeft : Integer;` in the API but not in the table; filled by code. |
| Localized | `localized name : String;` creates a `_texts` table for translations, read by user language. |
| Code lists | `sap.common.CodeList`, `Countries`, `Currencies`, `Languages` from `@sap/cds/common`. |
| Annotations | `@title`, `@mandatory`, `@readonly`, `@assert.range`, `@assert.format`, `@assert.target`, plus UI annotations. |
| CSV data | `db/data/<namespace>-<Entity>.csv`, semicolon or comma separated, loaded on deploy. |

**Association vs composition, the classic question:** an order **references** a customer (association: deleting the order keeps the customer). An order **owns** its items (composition: deleting the order deletes the items, and you can POST the order with its items in one request).

## Services and OData

A service is the API of your app: it decides which entities, fields and operations the outside world can see.

```cds
using sales from '../db/schema';

service CustomerService @(path: '/customers') {
  entity Orders as projection on sales.Orders excluding { modifiedBy };
  @readonly entity Customers as projection on sales.Customers;
}
```

| Concept | Remember |
| --- | --- |
| Projection | A view of an entity for one service; can rename, hide or add fields. |
| Multiple services | Different APIs on the same model, like `AdminService` and `CatalogService`, each with its own rules. |
| `@readonly` / `@insertonly` | Limit which operations a service entity allows. |
| `@cds.redirection.target` | Picks which projection to use when several point to the same entity. |
| Service path | Default `/odata/v4/<service-name-without-Service>`; change with `@path`. |
| Protocols | OData V4 by default; also `@protocol: 'rest'`, `'graphql'`, or OData V2 via an adapter. |
| `$metadata` | The machine-readable description of the service; Fiori reads it. |

**OData V4 query options**

| Option | Example | Does |
| --- | --- | --- |
| `$filter` | `$filter=status eq 'New' and amount gt 100` | Choose rows |
| `$select` | `$select=orderNo,status` | Choose columns |
| `$expand` | `$expand=items($select=product)` | Include related data |
| `$orderby` | `$orderby=createdAt desc` | Sort |
| `$top` / `$skip` | `$top=10&$skip=20` | Paging |
| `$count` | `$count=true` | Total number of rows |
| `$search` | `$search=Acme` | Free text search across `@cds.search` fields |
| `$apply` | `$apply=groupby((status),aggregate(amount with sum as total))` | Aggregation |

**HTTP methods:** GET reads, POST creates, PATCH updates part of a record, PUT replaces it, DELETE deletes. `$batch` bundles several requests in one call; Fiori uses it.

**Draft:** `@odata.draft.enabled` gives edit, save and discard with a work-in-progress copy. Keys become `(ID=…,IsActiveEntity=true|false)`. CAP stores drafts in separate draft tables and locks the active record while someone edits it.

## Custom logic: handlers, CQL, actions

CAP's generic handlers do CRUD for free; you add JavaScript only for business rules.

```js
import cds from '@sap/cds'
export default class OrderService extends cds.ApplicationService { init() {
  const { Orders } = this.entities
  this.before('CREATE', Orders, req => { if (!req.data.orderNo) req.error(400, 'Order number missing') })
  this.on('cancel', Orders, async req => { await UPDATE(req.subject).with({ status: 'Cancelled' }) })
  this.after('READ', Orders, orders => { /* add calculated values */ })
  return super.init()
}}
```

**Handler phases**

| Phase | Runs | Use for |
| --- | --- | --- |
| `before` | Before the work; several run in parallel | Validation, defaults, refusing |
| `on` | Does the work; first one wins, `next()` passes on | Replacing CRUD, actions, remote calls |
| `after` | After the work, gets the result | Calculated or virtual fields, logging |

**The req object**

| Part | Holds |
| --- | --- |
| `req.data` | Input payload or action parameters |
| `req.params` | Keys from the URL |
| `req.subject` | A CQN reference to the target record, for `SELECT.one.from(req.subject)` or `UPDATE(req.subject)` |
| `req.query` | The full CQN query of the request |
| `req.event` / `req.target` | Event name / target entity definition |
| `req.user` | `id`, `is(role)`, `attr` |
| `req.error()` / `req.reject()` / `req.warn()` / `req.info()` | Collect error and continue / stop at once / send messages |

**CQL (CDS Query Language) in JavaScript**

```js
await SELECT.from(Books).where({ stock: { '<': 5 } }).orderBy('title')
await SELECT.one.from(Books, id)                 // by key
await INSERT.into(Books).entries({ title: 'CAP' })
await UPDATE(Books, id).with({ stock: { '-=': 1 } })
await DELETE.from(Books).where({ stock: 0 })
await SELECT`count(*) as n`.from(Books)          // tagged template
```

CQL statements are CQN objects (Core Query Notation, plain JSON); `await` sends them to the database or a remote service.

**Actions vs functions**

|  | Action | Function |
| --- | --- | --- |
| Changes data? | Yes | No, read only |
| HTTP method | POST | GET |
| Bound | `entity X actions { action approve(); }` URL `X(ID)/Service.approve` | Same with `function` |
| Unbound | `action submit(...)` URL `/service/submit` | `/service/topItems()` |
| Wrong method | 405 Method Not Allowed | 405 |
| No handler | 501 Not Implemented | 501 |

**Events and messaging:** `this.emit('OrderCreated', data)` publishes an event; `srv.on('OrderCreated', ...)` subscribes. With `cds.requires.messaging` (SAP Event Mesh, Advanced Event Mesh, or file-based locally) events cross apps. The outbox (`cds.outboxed(srv)`) stores messages in the database so they are sent reliably after the transaction commits.

**Transactions:** every request runs in one transaction; an error rolls everything back. Use `cds.tx(async tx => ...)` for your own.

## Security: authentication and authorization

Authentication answers "who are you?"; authorization answers "what may you do?".

```cds
annotate ExpenseService with @(requires: 'authenticated-user');
annotate ExpenseService.Claims with @(restrict: [
  { grant: ['READ','WRITE'], to: 'Employee', where: 'createdBy = $user' },
  { grant: ['READ','approve'], to: 'Manager' },
  { grant: 'READ', to: 'Auditor' }
]);
```

| Concept | Remember |
| --- | --- |
| `@requires` | Who may use a service, entity or action at all: a role, `authenticated-user` or `any`. |
| `@restrict` | Fine rules: `grant` (events or actions), `to` (roles), `where` (which records). |
| Role-based | "Managers may approve." |
| Instance-based | `where: 'createdBy = $user'`: only your own records. Can also use user attributes: `where: 'country = $user.country'`. |
| `WRITE` / `*` | WRITE = CREATE, UPDATE, DELETE; `*` = everything. |
| Not granted | Forbidden by default once `@restrict` is present. |
| Mocked auth | Test users in `package.json` under `cds.requires.auth`; default in development. |
| XSUAA | BTP's OAuth 2.0 service; issues JWT tokens with scopes. |
| IAS | SAP Cloud Identity Services; the newer identity service, often with AMS for authorizations. |
| `xs-security.json` | Generated by `cds add xsuaa` (or `cds compile srv --to xsuaa`); lists scopes and role templates. |
| Role template, role collection | Template comes from your app; admin groups templates into role collections in the BTP cockpit and assigns them to users or groups. |
| Approuter | Handles login and forwards the JWT to the CAP backend. |
| `req.user` | `req.user.id`, `req.user.is('Manager')`, `req.user.attr.country`. |

**Status codes:** 401 = not logged in or bad token; 403 = logged in but not allowed; 404 = the `where` rule hides the record.

**Mocked to production:** annotations stay the same; only `cds.requires.auth.kind` changes from `mocked` to `xsuaa` or `ias` (automatic in the production profile).

## Fiori Elements with CAP

Fiori Elements builds standard UIs (list report, object page) from OData metadata plus annotations, so you write almost no UI code.

```cds
annotate TicketService.Tickets with @(
  UI.HeaderInfo: { TypeName: 'Ticket', TypeNamePlural: 'Tickets', Title: { Value: title } },
  UI.SelectionFields: [ status, priority ],
  UI.LineItem: [
    { Value: title },
    { Value: status, Criticality: statusCriticality },
    { $Type: 'UI.DataFieldForAction', Action: 'TicketService.close', Label: 'Close' }
  ],
  UI.Facets: [ { $Type: 'UI.ReferenceFacet', Label: 'Details', Target: '@UI.FieldGroup#Details' } ],
  UI.FieldGroup#Details: { Data: [ { Value: priority }, { Value: description } ] }
);
```

| Annotation | Controls |
| --- | --- |
| `UI.LineItem` | Table columns in the list report |
| `UI.SelectionFields` | Filter bar fields |
| `UI.HeaderInfo` | Object page title and type names |
| `UI.Facets` + `UI.FieldGroup` | Sections on the object page |
| `Criticality` | Colors: 1 red, 2 orange, 3 green, 0 neutral |
| `UI.DataFieldForAction` | Button for an action |
| `Common.ValueList` / `ValueListWithFixedValues` | Value help and dropdowns |
| `Common.Text` + `TextArrangement` | Show a name instead of an ID |
| `@UI.Hidden`, `@readonly`, `@mandatory` | Hide, lock, require fields |
| `@odata.draft.enabled` | Edit, Save, Discard buttons |

**Floorplans:** List Report + Object Page (most common), Worklist, Analytical List Page, Overview Page, Form Entry Object Page.

**Tools:** Fiori tools in BAS or VS Code: application generator, Page Map, guided development; the annotation file usually sits in `app/<app>/annotations.cds`. `cds watch` serves a Fiori preview link per entity.

**Deploy:** add the app with `cds add html5-repo` (or the generator's deployment config), deploy to the HTML5 Application Repository, then open it from SAP Build Work Zone.

## Integration with S/4HANA and other systems

CAP treats an external API like any other service: import its model, connect, and query it with CQL.

**Steps**

1. Download the EDMX (metadata) from SAP Business Accelerator Hub (api.sap.com).
2. `cds import API_BUSINESS_PARTNER.edmx` creates `srv/external/API_BUSINESS_PARTNER.csn` and adds it to `cds.requires`.
3. Expose only what you need: `entity BusinessPartners as projection on bp.A_BusinessPartner { key BusinessPartner as ID, BusinessPartnerFullName as name }`.
4. Delegate in code: `const S4 = await cds.connect.to('API_BUSINESS_PARTNER'); this.on('READ', BusinessPartners, req => S4.run(req.query))`.
5. Configure where it lives per profile.

```json
"cds": { "requires": { "API_BUSINESS_PARTNER": {
  "kind": "odata-v2", "model": "srv/external/API_BUSINESS_PARTNER",
  "[sandbox]": { "credentials": { "url": "https://sandbox.api.sap.com/s4hanacloud/sap/opu/odata/sap/API_BUSINESS_PARTNER" } },
  "[production]": { "credentials": { "destination": "S4HANA", "path": "/sap/opu/odata/sap/API_BUSINESS_PARTNER" } }
}}}
```

| Concept | Remember |
| --- | --- |
| Mocking | `cds watch` mocks imported services automatically; CSV files in `srv/external/data/` give test data. |
| `cds mock` | Runs a mock of the external service as a separate process. |
| Sandbox API key | Header `APIKey`; keep it in `.cdsrc-private.json` or env, never in git. |
| Destination service | Stores URL and credentials on BTP; CAP refers to it by name. |
| Connectivity + Cloud Connector | Reach on-premise systems securely from BTP (ProxyType OnPremise). |
| Principal propagation | The logged-in user is passed through to S/4HANA. |
| OData V2 vs V4 | S/4 APIs are often V2; set `kind: odata-v2`. CAP itself serves V4. |
| Expand across systems | A join between your DB and a remote service is not automatic; you combine them in code. |
| Resilience | Remote calls can fail or be slow: cache, batch lookups, handle errors. |
| Events from S/4 | S/4 business events through SAP Event Mesh; CAP subscribes with `srv.on('sap.s4.beh....', ...)`. |
| Other options | REST services (`kind: rest`), SAP Cloud SDK, SAP Integration Suite for heavier integration. |

## Database and deployment

| Database | When |
| --- | --- |
| SQLite (in memory or file) | Local development and tests; `cds watch` default |
| SAP HANA Cloud | Production on BTP; deployed through an HDI container |
| PostgreSQL | Supported alternative, for example on Kyma or other clouds |

**Key terms**

| Term | Meaning |
| --- | --- |
| HDI container | An isolated schema in HANA Cloud that holds your app's tables and views. |
| `db-deployer` | A small app that runs once at deploy time and creates or updates tables in the HDI container. |
| `cds deploy --to sqlite` | Writes the model and CSV data into a local SQLite file. |
| `cds deploy --to hana` / `cds watch --profile hybrid` | Deploy to HANA Cloud from BAS; run locally against a real cloud database. |
| `cds bind` | Links local runs to cloud service instances (hybrid testing). |
| MTA | Multi-Target Application: one `mta.yaml` describes all modules (srv, db-deployer, approuter, UI) and the BTP services they need. |
| `.mtar` | The archive built from `mta.yaml`. |
| Approuter | The single entry point: login, routing to backend and UI (standalone or managed by Work Zone). |

**Deploy to Cloud Foundry in four commands**

```bash
cds add hana,xsuaa,mta,approuter   # add production config
npm install
mbt build                          # creates mta_archives/<app>.mtar
cf deploy mta_archives/<app>_1.0.0.mtar
```

**Profiles:** development (default with `cds watch`) uses SQLite and mocked auth; production (`NODE_ENV=production` or `--production`) uses HANA and XSUAA/IAS automatically.

**Other runtimes:** Kyma (Kubernetes) with Helm charts via `cds add helm`; CF is the most common in interviews.

**Multitenancy (SaaS):** one app serves many customers (tenants), each with its own HDI container. `cds add multitenancy` adds the MTX sidecar that handles subscribe, upgrade and unsubscribe, with SaaS Provisioning registration. Extensibility lets tenants add their own fields.

**CI/CD:** SAP Continuous Integration and Delivery service or GitHub Actions; the pipeline runs build, tests and deploy.

## Testing and troubleshooting

**Testing**

```js
import cds from '@sap/cds'
const { GET, POST, expect } = cds.test(import.meta.dirname + '/..')

test('employee sees only own claims', async () => {
  const { data } = await GET('/odata/v4/expense/Claims', { auth: { username: 'anna', password: 'pass' } })
  expect(data.value.every(c => c.createdBy === 'anna')).to.be.true
})
```

- `cds.test()` starts the server in memory and gives HTTP helpers; run with Jest, Mocha or `node --test`.
- `.http` files (REST Client) are the quickest manual tests; each request can carry `Authorization: Basic user:pass`.
- Test service APIs directly too: `const srv = await cds.connect.to('OrderService'); await srv.run(SELECT.from('Orders'))`.

**Logging and debugging**

| Tool | Use |
| --- | --- |
| `const LOG = cds.log('orders'); LOG.info(...)` | Structured logs instead of `console.log` |
| `DEBUG=sql cds watch` | Show generated SQL |
| `cds.debug('...')` / `DEBUG=*` | Detailed framework logs |
| `cds watch --debug` or BAS debugger | Breakpoints in handlers |
| `cf logs <app> --recent` | Logs on Cloud Foundry |
| `cds compile srv --to sql` / `--to edmx` | See what CAP generates |

**Common errors**

| Error | Usual cause |
| --- | --- |
| 401 | No login or wrong user in mocked auth / missing token |
| 403 | Role missing in `@restrict` or role collection not assigned |
| 404 on an entity | Wrong service path or entity not exposed in the service |
| 405 | GET on an action, or POST on a function |
| 415 | Missing `Content-Type: application/json` |
| 501 "no handler for X" | Action declared but no `this.on('X')` handler |
| Handlers never run | `.js` file name does not match `.cds`, or `return super.init()` missing |
| "no such table" | Did not redeploy after changing the model, or wrong CSV file name |
| HDI deploy fails | Incompatible change (like changing a key) or missing HANA instance |
| Fiori shows no columns | `UI.LineItem` missing or annotation file not loaded |

## Top 30 interview questions with short answers

**Basics**

1. **What is CAP?** SAP's framework for building enterprise services and apps on BTP, using CDS for models and Node.js or Java for logic.
2. **Why use CAP instead of plain Express?** CRUD, OData, draft, auth, database and messaging come for free, so you write only business logic.
3. **What is CDS?** The language to define data models, services and annotations; CAP compiles it to SQL, EDMX and more.
4. **CDL, CQL, CQN, CSN?** CDL defines models, CQL queries data, CQN is a query as JSON, CSN is a compiled model as JSON.
5. **What does `cds watch` do?** Starts the server, uses SQLite in memory, loads CSV data, mocks auth and services, and restarts on every change.
6. **What are the main folders?** `db/` for the model and data, `srv/` for services and handlers, `app/` for UIs.
7. **What is a profile?** A named set of config, like development, production or hybrid, chosen with `--profile`.

**Modeling and services**

8. **Association vs composition?** An association references another entity; a composition owns its children (deep insert, cascade delete).
9. **Managed vs unmanaged association?** Managed lets CAP create the foreign key; unmanaged uses your own `on` condition.
10. **What are `cuid` and `managed`?** Aspects that add a UUID key, and created/modified fields filled automatically.
11. **What is a projection?** A view of an entity in a service that can rename, hide or add fields.
12. **Virtual vs calculated element?** Virtual is filled by your code; calculated is an expression CAP computes.
13. **How do you support translations?** Mark fields `localized`; CAP creates `_texts` tables and reads by user language.
14. **What is draft?** Work-in-progress editing for Fiori, enabled with `@odata.draft.enabled`.
15. **Which protocols does CAP serve?** OData V4 by default; also REST, GraphQL and OData V2 via adapter.

**Logic**

16. **Handler phases?** before (validate), on (do the work), after (enrich result).
17. **`req.error` vs `req.reject`?** `error` collects and fails at the end of the phase; `reject` stops at once.
18. **What is `req.subject`?** A reference to the record the URL points to, for use in `SELECT` or `UPDATE`.
19. **Action vs function?** Action changes data (POST); function only reads (GET).
20. **Bound vs unbound?** Bound belongs to one entity record; unbound belongs to the service.
21. **How do you call another service?** `const srv = await cds.connect.to('Name'); await srv.run(query)` or `srv.send(...)`.
22. **How do you emit events?** `this.emit('Name', data)`; subscribers use `srv.on('Name')`; messaging carries events across apps.

**Security, integration, deployment**

23. **Authentication vs authorization?** Who you are vs what you may do.
24. **`@requires` vs `@restrict`?** `@requires` checks a role for access; `@restrict` adds per-event and per-record rules.
25. **How do roles reach real users?** CDS roles become role templates in `xs-security.json`; admins put them in role collections and assign users.
26. **How do you consume an S/4HANA API?** `cds import` the EDMX, configure `cds.requires` with a destination, and delegate with `S4.run(req.query)`.
27. **How do you reach on-premise?** Destination service plus Connectivity service and Cloud Connector.
28. **What is an HDI container?** An isolated HANA schema with your app's database objects.
29. **How do you deploy?** `cds add hana,xsuaa,mta`, `mbt build`, `cf deploy` the `.mtar`.
30. **What is multitenancy in CAP?** One app for many customers, each with its own HDI container, managed by the MTX sidecar.

## Scenario questions

Interviewers like "how would you…" questions. Answer with the CAP feature first, then the detail.

| Scenario | Good answer |
| --- | --- |
| Stop orders when stock is too low. | A `before('CREATE')` handler reads stock with `SELECT.one` and calls `req.reject(409, ...)`. |
| Users must see only their own records. | `@restrict` with `where: 'createdBy = $user'`, using the `managed` aspect. |
| Show a customer name stored in S/4HANA next to local data. | Import the API, then an `after('READ')` handler looks up names, batching unique IDs to limit remote calls. |
| An approval button in Fiori. | Bound action `approve()`, a handler that runs `UPDATE(req.subject)`, `@restrict` granting it to Manager, and `UI.DataFieldForAction`. |
| Order header and items saved together. | Composition of many items; one deep POST, or draft in Fiori. |
| The app works locally but gets 403 on BTP. | The role collection is not assigned to the user, or the role name in `@restrict` differs from `xs-security.json`. |
| Remote API is slow. | Cache results, select only needed columns, batch lookups, add timeouts and error handling; consider events instead of polling. |
| Send an email after an order is created, even if the mail server is down. | Emit an event in an `after` handler and use the outbox, so it is sent reliably after commit. |
| Develop without access to S/4HANA. | `cds watch` mocks the imported service; add CSV files in `srv/external/data/`. Switch to the sandbox or real system with a profile. |
| Test against the real HANA database from BAS. | `cds bind` to the HDI instance, then `cds watch --profile hybrid`. |
| Same app for many customers. | Multitenancy with `cds add multitenancy`; each tenant gets its own HDI container. |
| Field must be one of a few values. | `enum` or `@assert.range`, plus `Common.ValueListWithFixedValues` for a dropdown. |
| Prevent a field being changed after creation. | `@readonly` for the API; `@Core.Immutable` to allow it only on create. |
| Expose the same data to admins and to public users differently. | Two services with different projections and different `@requires`. |

**Last tips for the interview**

- Say "CAP does it generically" before you talk about code: interviewers want to hear you avoid unnecessary code.
- Connect every feature to BTP: XSUAA for security, HANA Cloud for data, Destination for integration, Work Zone for UI.
- Have one project you can describe end to end: model, service, handler, security, Fiori, deploy.

🎓 Want live, instructor-led SAP CAP training? Request a live session here: https://forms.gle/ovT9EgQquL5ERW2c6
