# Workday Extend development

These preferences apply to **all** Workday Extend apps I work on (not any single project).

## Ask before guessing Workday syntax
When I am unsure about Workday Extend syntax, attribute names, valid enum values, built-in Workday Script function signatures, widget capabilities, or any Workday platform behavior, I STOP and ask a specific, targeted question before proceeding — I do not guess or invent syntax.

- **Why:** Workday Extend is a proprietary, niche platform thinly represented in training data. A confident wrong guess (nonexistent attributes/widgets/functions) wastes time. The user prefers a quick clarifying question over a plausible-but-wrong answer, and has explicitly offered to provide context on request.
- **How to apply:** Treat the existing code in the current repo as the source of truth for conventions and confirm against it first. If the repo doesn't settle it and I'm not certain, ask (e.g. "Does widget X support attribute Y in your Workday release?"). When I do proceed with any residual uncertainty, flag it explicitly rather than asserting.

## ID naming convention
Component IDs are always **camelCase** and **end with the component type**.

- Examples: `newProviderFieldSet` (a Field Set), `providerNameText` (a Text), `submitButton` (a Button), `providersGrid` (a Grid), `enrollmentSection` (a Section).
- Apply this whenever I create or rename an ID, and follow it when suggesting new components.

## Edit pages
- `pageType` is `"EDIT"` (counterpart to `"VIEW"`).
- Edit pages render framework **OK/Cancel** buttons automatically. Defining a custom `footer` (e.g. a "Powered By" richText) does **not** suppress those buttons — the footer content and the OK/Cancel buttons coexist.
- An editable `grid` can start empty with `"rows": "<% [] %>"`; users add rows via the `+` icon (`doNotAdd`/`doNotRemove` default to false).

## Widget types
- **Single boolean checkbox:** the widget `type` is `"checkBox"` (camelCase, capital B) — NOT `"checkbox"`. Binds to a boolean value, e.g. `"value": "<% row.isBrandNew ?? false %>"`. Distinct from `checkBoxList` (a multi-select list of options).
- **Single-select dropdown:** the widget `type` is `"dropdown"` (all lowercase) — NOT `"select"` or `"dropDown"` (`select` errors with "invalid tag"). Its choices come from an `instanceList` of `{ "id", "descriptor" }` objects. The selected value is an instance, so `myDropdown.value` returns the instance — use `.value.id` / `.value.descriptor` to get the string.
  ```json
  { "type": "dropdown", "id": "paymentMethodDropdown", "label": "Payment Option",
    "instanceList": [ { "id": "PAYROLL", "descriptor": "Payroll Deduction" } ] }
  ```
- **Text / number fields:** `"type": "text"` and `"type": "number"` are valid field widgets (with `id` + `value`). On VIEW pages they render read-only; on EDIT pages they're editable.
- **Button navigation:** a `button` navigates via a `taskReference` (app task) or `workdayTaskReference` (standard Workday task) attribute — mutually exclusive, one is REQUIRED. NOT a bare `taskId`. Shape: `{ "type": "button", "id": "backButton", "label": "Back", "taskReference": { "taskId": "Home" } }`.
- **Image / worker photo:** the widget `type` is `"image"`. Set `"userPhoto": true` to render a worker photo (no `value`/`url` needed) — useful as the avatar in a worker profile header. Sits as an ordinary sibling of `text` widgets inside a `fieldSet`.
  ```json
  { "type": "image", "id": "workerImage", "userPhoto": true }
  ```

## grid widget
Grid columns are NOT typed widgets placed directly in `columns`. Putting `"type": "text"`/`"number"` as the column type errors with "invalid tag" — those types are only valid INSIDE a `cellTemplate`.

- The grid declares `rowVariableName` (e.g. `"swagRow"`); cell bindings reference THAT alias, not `row` — e.g. `<% swagRow.name %>`.
- Each column is `{ "type": "column", "columnId": "...", "label": "...", "cellTemplate": { <widget> } }`.
- The actual widget (`text`, `number`, `dropdown`, `date`, `checkBox`, etc.) lives inside `cellTemplate` with its own `id` and `value`.
- **`required` goes on the `column`, NOT on the `cellTemplate` widget.** In a grid, mark a column required at the column level: `{ "type": "column", "columnId": "statusColumn", "label": "Status", "required": true, "cellTemplate": { ... } }`. (Outside a grid, `required` sits on the widget itself as usual.)
- Editable grid: start with `"rows": "<% [] %>"`; users add/remove rows via the +/trash icons. Set `isArrayOutBinding: true` to submit all rows as one outbound array.
- Read-only display grid: set `readOnly: true` on the grid.

```json
{
  "type": "grid",
  "id": "swagCatalogGrid",
  "label": "Available Swag",
  "rows": "<% [] %>",
  "rowVariableName": "swagRow",
  "columns": [
    {
      "type": "column",
      "columnId": "itemNameColumn",
      "label": "Item",
      "cellTemplate": { "type": "text", "id": "itemNameText", "value": "<% swagRow.name ?? '' %>" }
    },
    {
      "type": "column",
      "columnId": "priceColumn",
      "label": "Price",
      "cellTemplate": { "type": "number", "id": "priceNumber", "value": "<% swagRow.price %>" }
    }
  ]
}
```

## AMD task registration
Pages are registered in a `tasks` array in the **AMD**. Each entry: `{ "id": "<TaskId>", "routingPattern": "/...", "page": { "id": "<PageId>" } }`. The task `id` is what `taskId` references (hub `initialTask`/`navigationTasks`, flow `flowSteps`, button `taskReference`). The Home/landing task typically uses `"routingPattern": "/"`.

## PMD page naming convention
All PMD pages are named using **PascalCase** (e.g. `Dashboard`, `StartOETracker`, `PastOEs`). This applies to the page `id` and its task ID.

- This is distinct from component IDs *inside* a page, which are camelCase (see above).
- Apply this whenever I create or rename a PMD page, and follow it when suggesting new pages/tasks.

## Flow definitions

A **flow** is a sequence of edit pages that gives users a guided experience for a multistep transaction. A **flow step** represents each page (or task) in the flow. Use a flow to:
- Override the default navigation of an OK button on an edit page.
- Define conditional logic with page transitions.
- Define multiple navigation patterns using the same set of edit pages.

**Where defined:** Flows live in the `flowDefinitions` array in the **AMD** (not the PMD). Each flow references PMD/task pages by their `taskId`.

**Minimum steps:** A flow definition must have **at least 2 `flowSteps`** — a starting step and an ending step. A single-step flow fails validation ("Each flowDefinition must have at least 2 flowSteps"). For a one-edit-page interaction that should return to a landing page, make the edit page the `startsFlow` step and transition to an ending step whose `taskId` is the landing/view page (e.g. a Dashboard/hub or a conclude page).

**Navigation behavior:** By default, the OK button on an edit page navigates back to the previous page after submitting outbound endpoints. To redirect OK to a flow, reference the edit page's `taskId` in the flow's initial (starts-flow) step. When the flow reaches its last step, the user returns to the page they were on before the flow started.

**Flow step keys:**
- `id` — the flow step ID.
- `taskId` — the task ID of the edit/view page this step renders.
- `startsFlow: true` — marks the initial step.
- `endsFlow: true` — marks the final step.
- `transitions` — ordered list of possible next steps. Each transition has `order` (e.g. `"a"`, `"b"` — evaluated in order), `value` (the target step `id`), and `condition` (a Workday Script boolean expression). **The first `true` condition in `transitions` executes first.**

**View pages in a flow:** A view page (e.g. `orderConclude`) does NOT automatically show OK/Cancel buttons. Once the flow no longer controls navigation, add a button to navigate onward (e.g. to an Order History page).

**Example `flowDefinitions` (AMD):**
```json
"flowDefinitions": [
  {
    "id": "orderSubmitConcludeFlow",
    "flowSteps": [
      {
        "id": "orderStartStep",
        "taskId": "orderStart",
        "startsFlow": true,
        "transitions": [
          { "order": "a", "value": "orderSubmitStep", "condition": "<% flowVariables.isValid == true %>" },
          { "order": "b", "value": "orderStartStep",  "condition": "<% flowVariables.isValid == false %>" }
        ]
      },
      {
        "id": "orderSubmitStep",
        "taskId": "orderSubmit",
        "transitions": [
          { "order": "a", "value": "orderConcludeStep", "condition": "<% flowVariables.isConfirmed == true %>" },
          { "order": "b", "value": "orderStartStep",     "condition": "<% flowVariables.isConfirmed == false %>" }
        ]
      },
      {
        "id": "orderConcludeStep",
        "taskId": "orderConclude",
        "endsFlow": true
      }
    ]
  }
]
```

### flowVariables — passing values across pages in a flow

To send variable data into a flow, use an `outboundVariable` endpoint with `"variableScope": "flow"` in the **PMD**. To read it in a flow step's `condition`, reference the `flowVariables` app variable.

**Set a flow variable (PMD `outboundData`):**
```json
"outboundData": {
  "outboundEndPoints": [
    {
      "type": "outboundVariable",
      "variableScope": "flow",
      "name": "transitionOutboundVars",
      "values": [
        { "outboundPath": "isValid", "value": "<% !empty workerid.value %>" },
        { "outboundPath": "id",      "value": "<% workerid.value %>" }
      ]
    }
  ]
}
```

**Reference it in a flow step transition (AMD):** `"condition": "<% flowVariables.isValid == true %>"`.

## hub widget

A **hub** is a container widget: a left navigation pane plus task pages rendered on the right. Use it to create a centralized area with nav links to contextual tasks and related info.

**Rules & constraints:**
- **View pages ONLY.** The nav task pages must also be view pages — the hub does NOT support edit pages.
- The `hub` tag is the **only** tag in the presentation `body` of the view page.
- All task pages must be defined in the AMD.
- `queryParams` variables used in task pages must match the `parameters` sent from the hub.
- A task page can be used in only **1** hub — you cannot reuse the same `taskId` across multiple `hub`/`listDetailHub` widgets in multiple PMDs.
- `navigationTasks[]` max **20** items. `items[]` in a `group` max **5** (group items get **no** icons). `additionalLinks` max **20**.
- NOT supported embedded within a profile group or dashboard. Mobile: Android + iOS.

**Main attributes:**
- `label` (stringScript) — hub title at the top of the left nav pane.
- `icon` (string) — icon next to the title; use `wd-accent-*` (Hub Title Icons). Default is a task clipboard.
- `initialTask` (object, **required**) — default landing page shown on the right pane on load: `{ taskId, parameters }`. Does NOT appear as a nav link unless also added to `navigationTasks[]`.
- `navigationTasks[]` (**required**) — list of `item` and/or `group` objects:
  - `item`: `{ "type": "item", "label" (req), "task": { "taskId" (req), "parameters" }, "icon" (wd-icon-*), "render" (booleanScript, default true) }`.
  - `group`: `{ "type": "group", "label" (req, has expand/collapse arrow), "icon" (wd-icon-*), "render", "items": [ up to 5 items ] }`.
- `additionalLinks` (optional) — section at the bottom (separated by a line): `{ "title", "links": [...] }`. Each link is either:
  - `{ "type": "external", "label" (req), "url" (req) }` — opens an external URL outside the hub.
  - `{ "type": "task", "label" (req), "task": { "taskId", "parameters" } }` — opens a task page **outside** the hub.

`parameters` is `Map<String, StringScript>` (values support PMD scripting); the task page reads them via `queryParams`.

**Minimal shape (PMD `body`):**
```json
{
  "type": "hub",
  "label": "<% currentWorker.descriptor + ' Hub' %>",
  "icon": "wd-accent-award-medal",
  "initialTask": {
    "taskId": "hubTaskOne",
    "parameters": { "pageTitle": "Job Title", "field1": "<% currentWorker.primaryJob.businessTitle %>" }
  },
  "navigationTasks": [
    {
      "type": "item",
      "label": "Job Title",
      "task": { "taskId": "hubTaskOne", "parameters": { "pageTitle": "Job Title" } }
    },
    {
      "type": "group",
      "label": "Worker Details",
      "icon": "wd-icon-folder-close",
      "items": [
        {
          "type": "item",
          "label": "Skills",
          "task": { "taskId": "hubTaskTwo", "parameters": { "workerId": "<% currentWorker.id %>" } }
        }
      ]
    }
  ],
  "additionalLinks": {
    "title": "Suggested Links",
    "links": [
      { "type": "external", "label": "<% 'Workday' %>", "url": "<% 'https://www.workday.com' %>" }
    ]
  }
}
```

## Submitting data (outbound endpoints)

Outbound endpoints are REST APIs that add/update/delete data. When the user clicks **OK** on an edit page, Presentation Components build the JSON request body from widget values and submit it. Defined in the PMD's `outboundData.outboundEndPoints[]`.

**Required fields per endpoint:**
- `url` — relative REST API URL. Default HTTP method is **POST**; override with `httpMethod` (PATCH/PUT/DELETE/etc.).
- `baseUrlType` — a `dataProvider` key defined in the AMD.
- `name` — reference name used to bind field values to this endpoint.
- `authType` — the `authType` id from `authTypes[]` in the SMD. Workday REST APIs use `sso` (default).

**Two tag types in `outboundEndPoints[]`** (edit pages only; URL protocol must be HTTP/HTTPS):
- `outboundDataURI` — the **default** tag (no `type` needed). A REST outbound endpoint invoked on submit.
- `outboundVariable` — requires `"type": "outboundVariable"`. Persists variables sent to other pages (e.g. `"variableScope": "flow"` with `values[]` of `{ outboundPath, value }`). See the Flow definitions section for `flowVariables`.

```json
"outboundData": {
  "outboundEndPoints": [
    { "name": "submitRequisition", "url": "/requisitions", "authType": "sso", "baseUrlType": "workday-procurement" },
    {
      "type": "outboundVariable",
      "name": "outboundFlowVariableExample",
      "variableScope": "flow",
      "values": [ { "outboundPath": "someFlowVariable", "value": "<% someWidget.value %>" } ]
    }
  ]
}
```

**Key behaviors & constraints:**
- **Only the OK button** (or an `editButtonBar` submit button) can call outbound endpoints. A PMD script's `invoke` method CANNOT — but `invoke` can call a deferred inbound endpoint (e.g. an Orchestration for extra processing).
- Endpoints are invoked **in the order listed** in `outboundEndPoints[]`.
- `exclude` — a boolean binding; if true, the endpoint is skipped (conditional invocation).
- **No rollback:** if an endpoint fails, updates from preceding outbound requests are NOT rolled back.
- **Timeouts:** 24s per endpoint; 60s per request (initial loads + submissions + remote validations combined). Limit endpoint count and use performant endpoints.
- Use the `onSend` event to construct/manipulate the request body; `onMultiPartSend` for multipart.
- `responseErrorDetail` (on `outboundData`) maps endpoint errors into a page error message: `{ "errorSummary": "<% error %>", "errors": "<% errors.map(item => { item.error }); %>" }`.

**Binding widget values → request body:**
- `valueOutBinding` on a tag — format `outboundEndpointName.fieldPath`. The combined `valueOutBinding` data forms the JSON body. List notation: `outboundEndpointName.listName[].fieldPath` (or `[0]` to target a specific item).
- `valuesOut[]` on a tag — bind one widget to **multiple** fields/endpoints, or split a multi-part value (currency, date). Each item: `{ "value": "<% ... %>", "valueOutBinding": "..." }`.
- `values[]` on the **endpoint** — specify all outbound data on the endpoint itself instead of per-tag. Each item: `{ "outboundPath": "fieldPath", "value": "<% ... %>" }`. Useful for submitting data sourced from *another* endpoint.
- Grid → array: set the grid's `isArrayOutBinding` to true to submit all rows as one outbound array.
- Hidden tags carry values the user doesn't edit (e.g. ids from an inbound endpoint) into the request body.

**Example — conditional update vs delete on the same resource:**
```json
"outboundData": {
  "outboundEndPoints": [
    {
      "name": "updatePayInput",
      "baseUrlType": "workday-payroll",
      "url": "<% 'payrollInputs/' + queryParams.donationId %>",
      "httpMethod": "PATCH",
      "authType": "sso",
      "exclude": "<% !payInput.usedInCompletedResult %>"
    },
    {
      "name": "deletePayInput",
      "baseUrlType": "workday-payroll",
      "url": "<% 'payrollInputs/' + queryParams.donationId %>",
      "httpMethod": "DELETE",
      "authType": "sso",
      "exclude": "<% payInput.usedInCompletedResult %>"
    }
  ],
  "responseErrorDetail": {
    "errorSummary": "<% error %>",
    "errors": "<% errors.map(item => { item.error }); %>"
  }
}
```

**Example — `values[]` on the endpoint (also pulling from other endpoints):**
```json
{
  "name": "charity",
  "baseUrlType": "app",
  "url": "charities",
  "httpMethod": "POST",
  "authType": "sso",
  "values": [
    { "outboundPath": "name", "value": "<% name.value %>" },
    { "outboundPath": "minDonationAmount.value", "value": "<% minAmount.value %>" },
    { "outboundPath": "createdBy.id", "value": "<% worker.id %>" },
    { "outboundPath": "image.id", "value": "<% (!empty(charityImage.id)) ? charityImage.id : null %>" }
  ]
}
```

**Multipart data:** `valueOutBinding` format is `<outboundEndpointName>:<formDataName>:<jsonFieldName>` (nested field paths allowed). Manipulate parts in `onMultiPartSend` via `self.data` (add a `<key, map|array>` part, set `self.data.<part>.<field>`, or reconstruct to remove parts; form-data name can't be `metadata`).

**Multipart with attachments (`fileUploader` `valueOutBinding` format):**
- Workday REST API attachment-only → `outboundEndpointName`.
- Workday multipart (attachment + JSON) → `outboundEndpointName:filePartName`.
- Model attachment object REST API → `outboundEndpointName:defaultCollectionName`.
- Third-party multipart → `outboundEndpointName:filePartName`.

**Submitting unchanged data:** Extend skips PUT endpoints when nothing changed. Set the endpoint's `allowPutForUnchangedData: true` to force submission (important when building the body in `onSend`/`onMultiPartSend`).

**Customizing the edit page buttons:**
- Edit pages auto-render **OK** (submits + navigates) and **Cancel** (no submit, navigates back; flow variables persist).
- `standardEditButtonsHidden` (on `presentation`) — booleanScript to hide OK/Cancel conditionally.
- `editButtonBar` — override OK and add more **submit** buttons (e.g. `dropdownEditButton` with an `instanceList`); cannot customize Cancel. Define navigation via AMD flow definitions. Persist a dropdown selection as a flow variable using the dropDown's id.
- `cancelOverride` (on `presentation`) — override Cancel navigation; supports `parameters`/`parameterBindings`. E.g. `"cancelOverride": { "taskId": "home" }`.

## cardContainer & Page Configuration Cards

A **Page Configuration Card** is defined in its own **`.card` file** — NOT inline in the PMD. Its `id` **must match the card file name**. In App Builder they live under **Page Configurations > Cards** in the Components panel.

A Page Configuration Card's `body.type` is one of: **`chartCard`**, **`imageCard`**, **`listCard`**, **`pillCard`**, **`simpleCard`**.

Do NOT confuse these with **Extend Cards** (a different thing — admins add those to Workday entry points like the Home page, delivered Hubs, Search, and Journey; they live in the `Cards` section of the Components panel).

### cardContainer (the PMD side)

`cardContainer` is the container for a group of cards on a page. Multiple cards create a dashboard-like experience.

- **`card` tags are ONLY legal inside a `cardContainer`.** You cannot place a `card` directly in a page body.
- Every `card` must reference a Page Configuration Card via `cardId`, and may pass `parameters` for dynamic content (this is what lets one card template be reused across pages with different contexts).
- Valid on **view AND edit** pages. Mobile: Android + iOS.
- Static cards **always render above** dynamic cards when both are present.

**Attributes:**
- `layout` (string) — `grid` (**default**, rows of equal height) or `masonry` (columns; content determines card height).
- `cards` (card array) — static collection; array order = display order. Each entry: `{ "type": "card", "cardId" (req), "parameters" (mapScript), "render" (binding-Boolean) }`.
- `dynamicCards` (object) — renders a card per item in a list:
  - `on` (listScript, **req**) — script/endpoint returning the data list.
  - `as` (string, **req**) — item variable name (alphanumeric + underscore only).
  - `template` (object, **req**) — `{ "cardId" (stringScript, req — may be static OR dynamic), "parameters" (mapScript) }`.
- `id` (binding-String) — must start with a letter, unique per page. To be referenceable by other widgets, define it BEFORE them (otherwise "Undefined variable" on validation). Recommended for debugging.
- `render` (binding-Boolean, default true) — unlike `visible`, when false other tags **cannot reference** this tag on edit pages.

`parameters` is a **mapScript** and supports only: `boolean`, `date`, `list`, `map`, `number`, `set`, `string`.

```json
{
  "type": "cardContainer",
  "id": "statusCardContainer",
  "layout": "grid",
  "cards": [
    {
      "type": "card",
      "cardId": "simpleStatusCard",
      "parameters": "<% { 'title': 'Good news!', 'icon': 'wd-accent-fun', 'indicatorLabel': 'Success', 'indicatorColor': 'green', 'body': 'Your request was submitted.' } %>"
    }
  ],
  "dynamicCards": {
    "on": "<% helpCases.data %>",
    "as": "item",
    "template": {
      "cardId": "caseCard",
      "parameters": "<% { 'title': item.title, 'subtitle': item.caseID, 'body': item.detailedMessage, 'id': item.id } %>"
    }
  }
}
```

### simpleCard (the .card file side)

File `simpleStatusCard.card` → `"id": "simpleStatusCard"`. Set `body.type` to `"simpleCard"`.

**Attributes:**
- `id` (string, **req**) — must match the card file name.
- `parameters` (string array) — parameter names the PMD may pass in. Reference them bare in scripts (`<% title %>`), not via a prefix. Use `?: null` for optional ones (e.g. `"icon": "<% icon ?: null %>"`).
- `header` (object) — `title` (stringScript, bold, top of card), `subtitle` (stringScript, below title), `icon` (string — must be a **`wd-accent-*`** icon), `indicator` (object: `label` stringScript + `color`).
  - `indicator.color` valid values: **`blue`, `gray`, `green`, `orange`, `red`, `transparent`**.
- `body` (object, **req**) — `type` (**req**, `"simpleCard"`) and `value` (stringScript, **req**) = the body content.
- `footer` (TaskButton array) — up to **5** links. Each: `label` (stringScript, **req**) plus **exactly one** of:
  - `taskReference` — `{ "taskId", "parameterBindings": { ... } }` to navigate to an app task.
  - `workdayTaskReference` — `{ "wid": "..." }` for a Workday-delivered task/report (use its Integration ID).
  - `url` (dynamicBinding-String) — external link.
  - These three are **mutually exclusive**.

```json
{
  "id": "caseCard",
  "parameters": ["title", "subtitle", "icon", "indicator", "body", "id"],
  "header": {
    "title": "<% title %>",
    "subtitle": "<% subtitle %>",
    "icon": "<% icon ?: null %>",
    "indicator": {
      "label": "<% indicator.label ?: null %>",
      "color": "<% indicator.color ?: null %>"
    }
  },
  "body": {
    "type": "simpleCard",
    "value": "<% string:toString(body) %>"
  },
  "footer": [
    {
      "label": "Case Details",
      "taskReference": {
        "taskId": "caseDetails",
        "parameterBindings": { "caseWid": "<% id %>" }
      }
    }
  ]
}
```

### pillCard (the .card file side)

Displays a **group of buttons (pills)** on a card — e.g. a "Suggested Tasks" launcher. Set `body.type` to `"pillCard"`. Mobile: Android + iOS.

**Attributes:**
- `id` (string, **req**) — must match the card file name.
- `parameters` (string array) — parameter names the PMD may pass in (same semantics as `simpleCard`).
- `header` (object) — `title` (stringScript, bold, top of card), `subtitle` (stringScript), `icon` (string — **`wd-accent-*`** only), `indicator` (object: `label` stringScript + `color`).
  - `indicator.color` valid values: **`blue`, `gray`, `green`, `orange`, `red`, `transparent`**.
- `body` (object, **req**):
  - `type` (**req**) — `"pillCard"`.
  - `pills` (TaskButton array, **req**) — the buttons to display, **max 10**. Each pill: `label` (stringScript, **req**) + `taskReference` (**req**) — the page to navigate to. Note: unlike `footer`, a pill's ONLY navigation option is `taskReference` (no `url` / `workdayTaskReference`).
- `footer` (TaskButton array) — up to **5** links. Each: `label` (stringScript, **req**) plus **exactly one** of `taskReference` (`{ "taskId", "parameterBindings" }`), `workdayTaskReference` (`{ "wid": "..." }`), or `url` (dynamicBinding-String) — mutually exclusive.

```json
{
  "id": "pillCardExample",
  "header": { "title": "Suggested Tasks" },
  "body": {
    "type": "pillCard",
    "pills": [
      { "label": "Create charity", "taskReference": { "taskId": "createCharity" } },
      { "label": "Donate",         "taskReference": { "taskId": "donate" } },
      { "label": "View donations", "taskReference": { "taskId": "donations" } }
    ]
  },
  "footer": [
    { "label": "Home", "taskReference": { "taskId": "home" } }
  ]
}
```

Referenced from a PMD like any other card (`cardId` must match the card file name / `id`):

```json
{
  "type": "cardContainer",
  "layout": "grid",
  "cards": [
    { "type": "card", "cardId": "pillCardExample" }
  ]
}
```

Each `taskId` referenced by a pill must exist in the AMD `tasks` array.

## Workday Script built-in functions

This is the **complete** list of available built-in functions (user-provided, authoritative). If a function is not on this list, it does not exist — do not invent one. Two call styles appear:

- **Namespace/static form** — the target is the first argument: `list:filter(myList, 'type', 'functional')`, `regex:replace(text, regex, replacement)`.
- **Method form** — called on the value itself: `myList.size()`, `myString.trim()`, `myDate.plusDays(5)`.

Signatures below use the doc's own convention: an entry with no leading target argument (e.g. `filter (closure)`) is the method form; one whose first parameter is the collection (e.g. `filter (list, key, value)`) is the namespace form. Many functions offer both.

### IMPORTANT gaps to plan around
- **There is NO built-in `sum`, `average`, `count`, or `round`.** Aggregations must be hand-rolled — use `map` + `reduce` for a sum, then divide by `size()`. Guard against an empty list (`size() == 0`) since dividing by zero and reducing an empty list both fail.
- `number:` has `max`, `min`, `pow`, `sqrt`, `toBigDecimal`, and the two int converters — nothing else arithmetic.
- Closures use the `item => { expression }` form (e.g. `errors.map(item => { item.error })`).
- **This list is not a perfect match for every tenant/release.** `number:convertNumberToInt(...)` is on the list but throws `Unknown Function call` at runtime (observed 2026-08-10). Treat the `number:` namespace as suspect and prefer confirmed-in-use alternatives.
- **NUMBERS HAVE NO `.toString()`.** `toString ( )` exists for `date`, `list`, `map`, `set`, and `string` — the `number` namespace has only `convertNumberToInt`, `convertStringToInt`, `max`, `min`, `pow`, `sqrt`, `toBigDecimal`. Calling `.toString()` on a number throws, and inside a listCard row template that silently renders **zero rows** rather than an obvious error.
- **To stringify a number, concatenate onto an empty string: `'' + myNumber`.** Order matters — `String + Number` concatenates, but `Number + String` throws. So `'' + value + '%'` works while `value + '%'` does not.
- Once stringified this way you can chain string functions, which is how to truncate a decimal without any `number:` call: `('' + (a / b)).substringBefore('.')`.
- **Confirmed working in a real app:** `list:filter(list, key, value)`, `.map(closure)`, `.reduce(closure)`, `.size()`, `.substringBefore(separator)`, `date.toString()`, and `'' + number` for stringifying.

### bool
`all (expression1, expressionN)` · `any (expression1, expressionN)`

### bpfTaskHelper
`fetchTaskType (businessProcessTask)`

### converter
`booleanAsInt (expression)` · `booleanAsString (expression)`

### date
`add (precision, duration)` · `between (date2, precision)` · `after (date1, date2)` · `checkTodaysDate (timeZone)` · `createMonth (mm, yyyy)` · `createYear (yyyy)` · `extractValue (precision, date)` · `format (dateFormat)` · `formatDateWithTimeZones (date, inputTimeZone, inputDateTimeFormat, outputTimeZone, outputDateTimeFormat)` · `formatWithTimeZone (dateFormat, timeZone)` · `get (precision)` · `getDateTimeZone (timeZone)` · `getTodaysDate (timeZone)` · `getTodaysDateFormatted (timeZone, dateTimeFormat)` · `minusDays (days)` · `minusHours (hours)` · `minusMinutes (minutes)` · `minusMonths (months)` · `minusNanos (nanoseconds)` · `minusSeconds (seconds)` · `minusWeeks (weeks)` · `minusYears (years)` · `month ( )` · `now ( )` · `now (timeZoneString)` · `parse (dateString)` · `parse (dateString, dateFormat)` · `parse (dateString, dateFormat, timeZoneString)` · `parseDateString (dateString)` · `parseFormattedDateString (dateString, dateFormat)` · `plusDays (days)` · `plusHours (hours)` · `plusMinutes (minutes)` · `plusMonths (months)` · `plusNanos (nanoseconds)` · `plusSeconds (seconds)` · `plusWeeks (weeks)` · `plusYears (years)` · `timeAfter (date2)` · `toString ( )` · `withDayOfMonth (dayOfMonth)` · `withDayOfYear (dayOfYear)` · `withHour (hour)` · `withMinute (minute)` · `withMonth (month)` · `withNano (nanosecond)` · `withSecond (second)` · `withYear (year)` · `year ( )`

### file
`byteCountToDisplaySize (size)`

### fileType
`getFileType (fileName)`

### graph
`createId(id)` · `createId(id, idType)` · `createId(ids)` · `createId(ids, idType)` · `createIds(ids)` · `createIds(ids, idType)`

### grid
`getSubtotal (grid, columnId)`

### json
`asJSON (object)` · `attribute (key, value)` · `create (attribute1, attributeN)` · `parse (jsonString)` · `stringify (object)` · `query (source, jsonPath)` · `query (source, jsonPath, resultsAsList)`

### list
`add (element)` · `add (index, element)` · `add (list, object, index)` · `addAll (elements)` · `addAll (index, elements)` · `clear ( )` · `contains (element)` · `containsAll (elements)` · `createMapList (list, key)` · `createMapListWithKeys (list, keyList)` · `distinct ( )` · `emptyList ( )` · `exclude (list, key, value)` · `excludeEmptyAttribute (list, key)` · `excludeMultiple (list, key, comparisonList)` · `excludeRegex (list, regexValue)` · `filter (closure)` · `filter (closure(item, index))` · `filter (list, key, value)` · `filterEmptyAttribute (list, key)` · `filterMultiple (list, key, comparisonList)` · `filterRegex (list, regexValue)` · `find (closure)` · `first ( )` · `firstNonEmpty (param1, paramN)` · `flatten (list)` · `forEach (closure)` · `forEach (closure(item, index))` · `get (index)` · `indexOf (element)` · `indexOf (object, list)` · `isEmpty ( )` · `isList (object)` · `join ( )` · `join (list1, list2)` · `join (separator)` · `last ( )` · `lastIndexOf (element)` · `map (closure)` · `map (closure(item, index))` · `mapAttribute (list, key)` · `mapBeanAttribute (listOfBeans, beanProperty)` · `nonNull (list)` · `reduce (closure)` · `remove (element)` · `removeAll (elements)` · `retainAll (elements)` · `reverse ( )` · `set (index, element)` · `size ( )` · `sort ( )` · `sort (closure)` · `sort (list, key, ascendingOrder)` · `subList (fromIndex, toIndex)` · `toJson ( )` · `toList (value1, valueN)` · `toListIncludeNull (value1, valueN)` · `toMap (list, key)` · `toString ( )`

### map
`add (key, value)` · `addAll (anotherMap)` · `clear ( )` · `containsKey (key)` · `containsValue (value)` · `filter (closure)` · `forEach (closure)` · `get (key)` · `getKeys (map)` · `getObject (map, key)` · `getValues (map)` · `getValuesFromKeys (map, keys)` · `isEmpty ( )` · `keys ( )` · `map (closure)` · `mapKey (closure)` · `mapValue (closure)` · `put (key, value)` · `remove (key)` · `size ( )` · `toJson ( )` · `toString ( )` · `values ( )`

### number
`convertNumberToInt (number)` · `convertStringToInt (string)` · `max (number1, numberN)` · `min (number1, numberN)` · `pow (base, exponent)` · `sqrt (number)` · `toBigDecimal (arithmeticExpression, scale, roundingMode)`

### object
`defaultIfNull (object, default)` · `firstNonNull (object1, objectN)`

### regex
`find (text, regex)` · `match (text, regex)` · `replace (text, regex, replacement)` · `replaceOnce (text, regex, replacement)` · `split (text, regex)`

### set
`add (element)` · `addAll (elements)` · `clear ( )` · `contains (element)` · `containsAll (elements)` · `filter (closure)` · `find (closure)` · `forEach (closure)` · `isEmpty ( )` · `join ( )` · `join (separator)` · `map (closure)` · `reduce (closure)` · `remove (element)` · `size ( )` · `toJson ( )` · `toString ( )`

### string
`abbreviate (maxWidth)` · `capitalize ( )` · `concat (string1, string2)` · `contains (searchString)` · `contains (string, substring)` · `containsIgnoreCase (searchString)` · `defaultIfBlank (defaultString)` · `defaultIfEmpty (defaultString)` · `endsWith (suffix)` · `endsWithIgnoreCase (suffix)` · `formatListToString (separator, list)` · `formatMessage (messageWithParameters, orderedParameters)` · `fuzzyMatchIndex (list, comparisonValue)` · `fuzzyMatchString (list, comparisonValue)` · `fuzzyScore (query)` · `indexOf (searchString)` · `indexOf (substring, startIndex)` · `isAllLowerCase ( )` · `isAllUpperCase ( )` · `isBlank ( )` · `isNumber ( )` · `isNumeric ( )` · `isString (value)` · `join (array, separator)` · `join (string1, stringN)` · `lastIndexOf (searchString)` · `lastIndexOf (subString, fromIndex)` · `leftPad (size)` · `leftPad (size, padCharacter)` · `length ( )` · `lowerCase ( )` · `lowerCase (locale)` · `pathEncode ( )` · `remove (subString)` · `removeEnd (subString)` · `removeEndIgnoreCase (subString)` · `removeStart (subString)` · `removeStartIgnoreCase (subString)` · `replace (searchString, replacement)` · `replaceIgnoreCase (searchString, replacement)` · `replaceOnce (searchString, replacement)` · `replaceOnceIgnoreCase (searchString, replacement)` · `replaceSubstring (string, origSubstring, newSubstring)` · `reverse ( )` · `rightPad (size)` · `rightPad (size, padCharacter)` · `size ( )` · `split (separatorChars)` · `splitByRegex (regex)` · `startsWith (prefix)` · `startsWith (string, substring, startIndex)` · `startsWithIgnoreCase (prefix)` · `stripPrefix (prefix)` · `stripSuffix (suffix)` · `substring (startIndex)` · `substring (startIndex, endIndex)` · `substringAfter (separator)` · `substringAfterLast (separator)` · `substringBefore (separator)` · `substringBeforeLast (separator)` · `toDecimal ( )` · `toDecimal (scale, mode)` · `toInt ( )` · `toString ( )` · `trim ( )` · `trimToEmpty ( )` · `truncate (maxWidth)` · `truncate (offset, maxWidth)` · `uncapitalize ( )` · `upperCase ( )` · `upperCase (locale)` · `urlDecode ( )` · `urlEncode ( )` · `uuid ( )`

### validate
`match (regex, comparisonString)`

### Averaging a list (the hand-rolled pattern)

Since there is no `sum`/`average`, average a numeric attribute like this — note the empty-list guard, which is required:

```
<% myList.size() == 0
     ? '0%'
     : (myList.map(item => { item.progress ?? 0 })
              .reduce((runningTotal, value) => { runningTotal + value })
        / myList.size()
       ).toString().substringBefore('.') + '%'
%>
```

`substringBefore('.')` truncates the decimal — it returns the whole string unchanged when there's no `.`, so it is safe whether the division yields `53.33` or a clean `100`. Do NOT reach for `number:convertNumberToInt` here; it throws `Unknown Function call`.

When comparing an average against exact boundaries (0 or 100), skip rounding entirely and compare the raw quotient — those cases are exact.

Keep percentages stored as **bare numbers** (`"progress": 85`) and append the `%` only at render time (`value.toString() + '%'`). Storing `"85%"` makes the value a display string and forces a `substringBefore('%')` + `toInt()` round-trip before any arithmetic.
