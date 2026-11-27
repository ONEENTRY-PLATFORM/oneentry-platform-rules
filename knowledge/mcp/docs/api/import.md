# Import

Bulk-loading content from a file, driven by an import template that maps the file's columns onto entity fields.

Import is the fastest way to create a lot of correct data and the fastest way to create a lot of wrong data. The difference is whether you tested with a small file first.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/api/index-attributes`

## Templates and sessions

An **import template** describes how a source file maps onto entities: which column feeds which attribute, how values are interpreted, what identifies an existing record.

An **import session** is one run of a template against one file. Sessions have a lifecycle you can read, which is how you find out what a run did.

```text
cms_api_search { "query": "import" }
```

## Template identifiers are unique

Creating a template whose identifier exists fails. Read the templates before creating one, and reuse an existing template rather than making a near-duplicate — two templates that differ by one column are how a wrong mapping gets used by accident.

An identifier may contain only lowercase Latin letters, digits and underscores (`catalog_daily_feed`). Anything else is refused with `400` naming `identifier`. The identifier is what a template run is started by, as a path segment, so derive it from the template name as a slug instead of passing the name itself. Templates saved before this rule may carry other characters; they keep their identifier on update, and a run addresses them with the identifier URL-encoded.

## Building a mapping

The mapping targets attributes, so you need the attribute set first: each attribute's type, its identity, and its marker. A column mapped onto an attribute of the wrong type either fails per row or stores something unusable.

Check specifically:

- **Numbers.** An empty cell is not zero. Decide what an empty cell should mean.
- **Choice attributes.** They store option ids; a column of human labels has to be mapped to ids.
- **Files and images.** Values are references to uploaded files, not paths in the source file.
- **Locales.** Imported content is locale-keyed like everything else. Know which locale a file is for.

→ `mcp/docs/api/attribute-types`

## Always test with a small file first

Run the template against a handful of rows before the full file. Then read the created entities **by id** and check every column landed where you expected, especially the attributes.

The trap this avoids is the silent one: an attribute mapping that produces empty values reports success on every row.

## Import is slow to become searchable

A large import lands in the entities immediately but becomes filterable and searchable progressively. On a big file that is minutes, not seconds.

So verify by sampling — read a few entities by id — rather than by refreshing a listing and concluding rows are missing.

→ `mcp/docs/api/index-attributes#bulk-changes-take-proportionally-longer`

## What happens when the file's columns differ from the template

A template remembers the header of every column it maps. When you run a spreadsheet or CSV template against a new file, the headers of that file are compared with the remembered ones before anything is imported:

- **Same headers, same positions.** The run starts as usual.
- **Same headers, other positions** (a column added in front, columns reordered). Each column is found by its header and the run starts; values still land in their own fields.
- **A remembered header is absent.** The run does not start. The call answers `400`, and `detail` names the file and the missing columns.

Renaming a column header in the source file therefore breaks the template on purpose. Send the file with the template's headers, or build a new template from the new file. Do not rename the template's columns by hand to make a run pass.

## Imports create records and records cost quota

Every imported row is a record against the instance's total limit. A doubled import does not merely duplicate content, it can exhaust the quota and make **all** further creates fail, including unrelated ones.

Before a large run, check the counters. After a failed run, find out what was created before deciding to run it again.

→ `mcp/docs/api/settings#the-record-limit-and-what-it-feels-like`

## Re-running an import

Whether a second run updates or duplicates depends on whether the template identifies existing records. If it does not, a re-run creates everything again.

What identifies an existing record is a flag on an attribute in the set: `isSku` or `isUniqueKey` marks the field rows are matched against. A set carrying neither has nothing to match on, so every run inserts.

Check the set for that flag first, then confirm the behaviour with a two-row test before re-running anything at scale.

Two kinds of row need a look before a run, because the match field cannot do its job for them:

- **A row with the match field empty is skipped.** It could never be matched on a later run, so importing it would add a new record every time. Fill the value in the file if the row matters.
- **A value repeated within one file yields one record.** The rows are applied in order, so the last one wins. The run still succeeds; the repetition is reported as a warning.

The finished job reports both, together with rows skipped for an empty name: `skipped` lists each row with a `reason` (`empty_unique_key`, `empty_name`, `out_of_scope`), `warnings` lists each repeated value (`duplicate_unique_key`) with the rows that carried it, and `skippedCount` / `warningsCount` give the totals. Row numbers count data rows from 1, the header row not included. A successful job with a non-zero `skippedCount` did not import every row.

## Number and price cells

A cell mapped onto a number attribute, or onto the attribute marked as the product price, is read as a number written the way spreadsheets write it. Spaces between digit groups are dropped, a single comma is the decimal separator, and when both a comma and a dot appear the right-most one is. So `100,50` imports as `100.50` and `1 200,00` as `1200.00`.

A cell that is not a number after that, such as `12 USD` or `abc`, and a negative price, is imported empty. It is never turned into `0`, because a price of zero sells the product for free. The run still succeeds and lists the cell in `warnings` with `reason` `invalid_number` or `negative_price`, its row, column header and original value. An empty number cell stays empty.

After a catalog run, read `warningsCount`: a non-zero value means some products have no price or an empty number field. Fix the cells in the file and run the import again.

## Starting an import needs the rights of what it writes

A run writes as the account that starts it, so besides the right to import it needs the rights the same write needs by hand. Without them the start answers `403` naming the missing permissions:

- **Catalog and content imports** need `catalog.products.create`. A template that matches existing records by a unique key also needs `catalog.products.update`. A run that only uploads additional files needs `catalog.products.update` alone.
- **Order imports** need `orders.storage.order.update`.
- **User imports** need `users.create`, and **form submission imports** need `forms.data.update`.

An unknown storage type is refused with `400`.

Scope applies too. A catalog-scoped account imports only into a category inside its scope, an order-scoped one only into a storage it may reach; otherwise `403`. A row whose unique key matches a product outside the catalog scope is not applied and is listed in `skipped` with `reason` `out_of_scope`.

→ `mcp/docs/api/admins-and-permissions`

## Rolling back an import needs the rights of what it removes

Deleting an entry from the import history rolls the run back: the records it created are removed, products with or without a category. Products it only updated stay, and a listed attribute set goes only once no product uses it. That call checks the caller against every section the entry holds records of, not just the right to import.

- **The delete right of each section.** An entry that created products needs the product delete right, one that created pages needs the page delete right, and so on for statuses, users, form submissions, orders and attribute sets. A missing right answers `403` with `PERMISSION_DENIED` and the `permission` it lacks. Nothing is removed.
- **The records must be in the caller's scope.** Pages must be in the caller's page scope, orders in a storage they may reach, form submissions in a form they may reach. A product is removed whole, so every category it sits in must be in the caller's catalog scope. One record outside scope answers `403` with `SCOPE_DENIED`, and nothing is removed.

So an account that may import is not automatically an account that may undo an import. Read the history entry first to see which sections it touches, and roll back with an account that holds the delete rights and scope for all of them.

The right to import does not let an account write into sections directly. Creating, changing and deleting records outside a run still needs the section's own rights.

→ `mcp/docs/api/admins-and-permissions`

## Reading what a session did

Read the session rather than inferring from the data. It tells you what was processed and what failed, which is the difference between "the file had bad rows" and "the mapping is wrong".

If a run reports failures, fix the template or the file. Do not re-run the same combination hoping for a different outcome.

## Common mistakes

- **Running the full file first.** Test with a few rows.
- **Trusting row counts as verification.** Read entities by id and check attributes.
- **Refreshing a listing to check progress.** Sample by id instead.
- **Re-running after a partial failure** without checking what already exists.
- **Mapping labels onto choice attributes.** They need ids.
- **Rolling back with an account that may only import.** The rollback needs the delete rights and scope of every section the run wrote to.
- **Renaming headers in a file a template runs on.** The run is refused with `400`; keep the headers or build a new template.

→ `mcp/docs/api/verification-recipes`
