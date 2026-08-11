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

## Imports create records and records cost quota

Every imported row is a record against the instance's total limit. A doubled import does not merely duplicate content, it can exhaust the quota and make **all** further creates fail, including unrelated ones.

Before a large run, check the counters. After a failed run, find out what was created before deciding to run it again.

→ `mcp/docs/api/settings#the-record-limit-and-what-it-feels-like`

## Re-running an import

Whether a second run updates or duplicates depends on whether the template identifies existing records. If it does not, a re-run creates everything again.

Establish which behaviour the template has — with a two-row test — before re-running anything at scale.

## Reading what a session did

Read the session rather than inferring from the data. It tells you what was processed and what failed, which is the difference between "the file had bad rows" and "the mapping is wrong".

If a run reports failures, fix the template or the file. Do not re-run the same combination hoping for a different outcome.

## Common mistakes

- **Running the full file first.** Test with a few rows.
- **Trusting row counts as verification.** Read entities by id and check attributes.
- **Refreshing a listing to check progress.** Sample by id instead.
- **Re-running after a partial failure** without checking what already exists.
- **Mapping labels onto choice attributes.** They need ids.

→ `mcp/docs/api/verification-recipes`
