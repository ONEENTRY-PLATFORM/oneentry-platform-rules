# Map of this documentation

Every document in this corpus, with the reason you would read it. When a search does not surface what you need, come here and pick by subject.

Start with `mcp/operating-rules`. Everything else is detail behind one of its rules.

→ `mcp/operating-rules` · `mcp/docs/server/getting-started`

## Read these first

| docId | Read it when |
|---|---|
| `mcp/operating-rules` | Before your first write. The short rules that decide whether a payload works |
| `mcp/docs/server/getting-started` | Setting the server up, or making your first three calls |
| `mcp/docs/api/baseline-data` | Before creating anything. What already exists on every instance, and what does not |
| `mcp/docs/api/silent-no-ops` | Before reporting any write as done. The operations that answer success and do nothing |
| `mcp/docs/server/payload-conventions` | Building any request body |

## Running the server

| docId | Read it when |
|---|---|
| `mcp/docs/server/configuration` | You need a flag, an environment variable, or the precedence between them |
| `mcp/docs/server/remote-mode` | Hosting one server for many agents over HTTP |
| `mcp/docs/server/authentication` | Credentials are rejected, or you need to know how identity works |
| `mcp/docs/server/api-catalog` | Searches find nothing, or `cms_whoami` shows catalog warnings |
| `mcp/docs/server/knowledge-subsystem` | The documentation is degraded, or you want to run against a local copy |

## Write safety

| docId | Read it when |
|---|---|
| `mcp/docs/server/allow-levels` | A call was refused before anything was sent |
| `mcp/docs/server/confirm-and-dry-run` | Previewing a mutation, or handling a confirm token |
| `mcp/docs/server/audit-log` | You need to know what is recorded, or to read the log |
| `mcp/docs/api/version-history` | Taking a snapshot of a record, or rolling one back to an earlier version |

## The tools

| docId | Read it when |
|---|---|
| `mcp/docs/server/cms-guide-and-whoami` | Working out where you are and what you may do |
| `mcp/docs/server/cms-docs-search-and-read` | Searching this corpus, or paging a long document |
| `mcp/docs/server/cms-api-search` | Finding an operation |
| `mcp/docs/server/cms-api-describe` | Getting an operation's parameters and body shape |
| `mcp/docs/server/cms-api-call` | Actually calling one |
| `mcp/docs/server/cms-upload-file` | Putting a file into the instance, from disk or from an address |
| `mcp/docs/server/response-shaping` | A response came back truncated or with values stripped |

## When something goes wrong

| docId | Read it when |
|---|---|
| `mcp/docs/server/errors-and-refusals` | You have an error message and need the next action |
| `mcp/docs/server/errors-startup` | The server will not start, or every call fails the same way |
| `mcp/docs/server/glossary` | A term in a payload or a message is unfamiliar |

## Working end to end

| docId | Read it when |
|---|---|
| `mcp/docs/server/agent-workflows` | You want the whole sequence for a common task |
| `mcp/docs/api/verification-recipes` | You need to prove a change worked |
| `mcp/docs/api/bulk-content-migration` | The task is hundreds of writes rather than one |
| `mcp/docs/api/silent-no-ops` | A write answered 200 or 201 and you are about to call it done |
| `mcp/docs/server/authoring-knowledge` | You are adding to or correcting this documentation |

## Content structure

| docId | Read it when |
|---|---|
| `mcp/docs/api/locales` | Writing localized content, or content is blank in a language |
| `mcp/docs/api/content-modelling` | Deciding where content goes, before the first write |
| `mcp/docs/api/attribute-sets` | Building any payload with attribute values |
| `mcp/docs/api/attribute-types` | A value came back in a shape you did not expect |
| `mcp/docs/api/list-options-and-extra-values` | A field a visitor picks from, or an option that needs an image or a colour |
| `mcp/docs/api/general-types` | Creating an entity that needs a type |
| `mcp/docs/api/pages` | Working with the page tree |
| `mcp/docs/api/page-listing-filters` | Listing chosen pages by id, or narrowing a level of the tree by type or condition |
| `mcp/docs/api/blocks` | Creating or attaching blocks |
| `mcp/docs/api/block-types` | A dynamic block is empty, or you are configuring one |
| `mcp/docs/api/similar-product-blocks` | Configuring a block that shows products similar to another |
| `mcp/docs/api/menus` | Building navigation, or a menu reorganised itself |
| `mcp/docs/api/templates-and-previews` | Changing how something renders |
| `mcp/docs/api/files-and-uploads` | Uploading files or referencing them from attributes |
| `mcp/docs/api/content-api-reads` | A public read refuses, lags, or serves nothing |

## Catalogue

| docId | Read it when |
|---|---|
| `mcp/docs/api/products` | Creating, updating or listing products |
| `mcp/docs/api/product-statuses` | Setting whether a product is available |
| `mcp/docs/api/product-relations` | Related, similar or recommended products |
| `mcp/docs/api/filters` | Narrowing a listing, or configuring facets |
| `mcp/docs/api/index-attributes` | A filter finds nothing, or a new value is missing from a list |
| `mcp/docs/api/global-search` | Locating an entity by a word a human used, or building a site search on the public one |
| `mcp/docs/api/file-content-search` | Finding a document by words written inside it, or a document search that returns nothing |
| `mcp/docs/api/search-query-correction` | Correcting typos in a public search query, or a search that answered with a corrected query |

## Commerce

| docId | Read it when |
|---|---|
| `mcp/docs/api/orders` | Reading or changing orders |
| `mcp/docs/api/order-statuses` | Defining order statuses, and which axis one belongs to |
| `mcp/docs/api/order-status-triggers` | Moving an order between states: triggers, status maps, and why there is no transition |
| `mcp/docs/api/payments` | Payment accounts, sessions or refunds |
| `mcp/docs/api/discounts` | Discounts, coupons and how they combine |
| `mcp/docs/api/subscriptions` | Recurring plans and customer enrolments |

## Forms and people

| docId | Read it when |
|---|---|
| `mcp/docs/api/forms-and-form-data` | Defining a form, or binding it so it accepts anything |
| `mcp/docs/api/form-submissions` | Listing, filtering or reading what visitors submitted |
| `mcp/docs/api/form-submission-scope-filters` | Submissions about a category, a branch of pages or a group of users |
| `mcp/docs/api/sensitive-form-fields` | Marking a form field sensitive, or a submission answering `value: null` for a filled field |
| `mcp/docs/api/captcha-fields` | A form has a captcha field and submissions are refused |
| `mcp/docs/api/rating-forms-and-reviews` | Reviews, star ratings, or importing either |
| `mcp/docs/api/collections` | Keeping a list of rows outside the page tree, or clearing one |
| `mcp/docs/api/users-and-groups` | A Content API route returns a permission error |
| `mcp/docs/api/content-api-permission-rules` | A public list is short, or a public write succeeded that should not have |
| `mcp/docs/api/content-api-sign-in-and-cart` | Building the account or cart pages of a site |
| `mcp/docs/api/social-sign-in` | Setting up Google, Apple, VK or another network to sign in with |
| `mcp/docs/api/password-reset` | A visitor cannot sign in and needs a new password |
| `mcp/docs/api/admins-and-permissions` | An operation was refused for lack of a permission |
| `mcp/docs/api/admin-page-scope` | Delegating one branch of the page tree, or a page write refused for an admin who holds the key |
| `mcp/docs/api/admin-account-ownership` | Changing another administrator's login, e-mail, password or permissions, or a write on the project owner refused with 403 |
| `mcp/docs/api/admin-sign-in` | Administrator sign-in answered a challenge instead of tokens, or a sign-in code is refused |

## Instance configuration

| docId | Read it when |
|---|---|
| `mcp/docs/api/modules` | Explaining why an admin section is missing |
| `mcp/docs/api/journal` | Finding who changed something, or clearing the instance's own record of it |
| `mcp/docs/api/content-api-journal` | Showing what the public API has been asked for, or chasing an error a site reports |
| `mcp/docs/api/events` | Sending a mail or a push when something happens |
| `mcp/docs/api/workflows` | A requirement shaped like a schedule, or a reaction no endpoint offers |
| `mcp/docs/api/workflow-nodes` | Writing the nodes of a flow, and the configuration the catalogue does not declare |
| `mcp/docs/api/settings` | Instance-wide settings, limits and quota |
| `mcp/docs/api/environments` | A second set of data on the same instance, and why your writes do not reach it |
| `mcp/docs/api/environment-markers` | Pointing a client at another data environment without editing the client |
| `mcp/docs/api/environment-transfer` | A transfer that carries several kinds, overwrites conflicts, deletes in the target, or meets a duplicate there |
| `mcp/docs/api/import` | Bulk-loading content from a file |
| `mcp/docs/api/ai-gateway` | The platform's own model-calling feature |
| `mcp/docs/api/interface-modules` | Creating or changing a simplified copy of an admin section for chosen admins |
| `mcp/docs/api/interface-module-scopes` | What a module's scope and saved filter accept, per base |
| `mcp/docs/api/interface-module-data` | Reading or exporting what an interface module shows |
| `mcp/docs/api/ui-component-configs` | Changing how an admin panel table, form or record page looks, or giving an admin a screen template |

## Answers about permissions and refusals

| Question | Go to |
|---|---|
| Why was my call refused with nothing sent | `mcp/docs/server/allow-levels#a-level-refusal-happens-before-authentication` |
| Why does every public read answer 401 | `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header` |
| Why does a public read answer 503 | `mcp/docs/api/content-api-reads#what-a-503-on-a-public-read-is-telling-you` |
| Why does a public call answer 405 with an HTML page | `mcp/docs/api/content-api-reads#a-405-with-an-html-body-means-the-instance-is-overloaded` |
| Why does a public read answer 429 | `mcp/docs/api/content-api-reads#what-a-429-on-a-public-read-is-telling-you` |
| Why does sign-in answer 429 | `mcp/docs/server/authentication#why-sign-in-answers-429` |
| Why does password recovery answer 429 | `mcp/docs/server/authentication#why-sign-in-answers-429` |
| Why did sign-in answer mfaRequired instead of a token | `mcp/docs/api/admin-sign-in#sign-in-answers-a-challenge-instead-of-tokens` |
| Why is my sign-in code refused | `mcp/docs/api/admin-sign-in#finish-the-sign-in-with-the-e-mailed-code` |
| Why does the sign-in code call answer 429 | `mcp/docs/api/admin-sign-in#too-many-attempts-answers-429` |
| Why does sign-in answer 503 with no challenge | `mcp/docs/api/admin-sign-in#when-the-code-cannot-be-sent` |
| How do I reset a forgotten admin password | `mcp/docs/api/admin-sign-in#resetting-a-forgotten-admin-password` |
| Why is the server unauthenticated with the right login and password | `mcp/docs/server/authentication#an-instance-with-a-second-factor-needs-the-token` |
| Why does a read say permissions are temporarily unavailable | `mcp/docs/api/content-api-reads#what-a-503-on-a-public-read-is-telling-you` |
| Why can I not list the submissions of a form | `mcp/docs/api/form-submissions#both-read-routes-need-the-forms-data-read-permission` |
| Why did an operation that worked start answering 403 | `mcp/docs/api/admins-and-permissions#a-key-can-exist-without-any-admin-holding-it` |
| Why is an interface modules call refused with 403 | `mcp/docs/api/interface-modules#which-permission-keys-manage-interface-modules` |
| Why is my upload or file delete refused with 403 | `mcp/docs/api/files-and-uploads#admin-uploads-and-deletes-need-file-permissions` |
| Why is the document search refused with 403 | `mcp/docs/api/file-content-search#two-permissions-nobody-holds-yet` |
| Why does a flow node refuse every method I send | `mcp/docs/api/workflow-nodes#configure-the-cms-api-node` |
| Why is a flow's event output example empty | `mcp/docs/api/workflow-nodes#configure-the-cms-events-node` |
| Why was my admin permission write refused with 400 | `mcp/docs/api/admins-and-permissions#an-unrecognised-permission-key-is-refused-with-400` |
| Which permission values are not booleans | `mcp/docs/api/admins-and-permissions#a-recognised-key-with-the-wrong-value-is-a-different-400` |
| Why does creating an admin answer 405 | `mcp/docs/api/admins-and-permissions#creating-an-admin-can-answer-405` |
| Why can I not edit the project owner account | `mcp/docs/api/admin-account-ownership#nobody-but-the-owner-changes-the-owner-account` |
| How do I set a password for another admin | `mcp/docs/api/admin-account-ownership#no-operation-sets-another-admin-password` |
| Why did an admin e-mail change not apply | `mcp/docs/api/admin-account-ownership#a-new-e-mail-waits-for-confirmation` |
| Why can I not grant a permission I was asked to give | `mcp/docs/api/admin-account-ownership#who-may-change-permissions` |
| How do I delegate one section of the site to an editor | `mcp/docs/api/admin-page-scope#delegating-a-section-of-the-site-to-an-editor` |
| Why is my page create or delete refused with 403 | `mcp/docs/api/admin-page-scope#what-a-refusal-tells-you` |
| How do I lift a page scope restriction | `mcp/docs/api/admin-page-scope#there-is-no-empty-list` |
| Which permission does rolling back a record need | `mcp/docs/api/version-history#which-permission-each-call-needs` |
| Why does a version history call answer 400 | `mcp/docs/api/version-history#which-records-have-a-version-history` |
| Can an export permission be granted | `mcp/docs/api/admins-and-permissions#exporting-data-has-its-own-permissions` |
| Why is creating or deleting a data environment refused with 403 | `mcp/docs/api/admins-and-permissions#managing-data-environments-has-its-own-permission` |
| Why does naming an environment answer 400 before my token is checked | `mcp/docs/api/environments#what-a-wrong-environment-name-answers` |
| Which data environment do my calls reach | `mcp/docs/api/environments#which-data-set-your-own-calls-reach` |
| How do I point a client at a different data environment | `mcp/docs/api/environment-markers#a-marker-is-a-second-way-to-name-an-environment` |
| Why does renaming an environment marker change nothing | `mcp/docs/api/environment-markers#why-renaming-a-marker-does-nothing` |
| Why did a transfer answer 201 and leave the target page as it was | `mcp/docs/api/environment-transfer#a-duplicate-without-a-resolution-is-not-written` |
| Why does a transfer answer 400 for a type listed twice | `mcp/docs/api/environment-transfer#each-kind-may-appear-only-once-in-items` |
| Why did a delete-mode transfer remove more than the ids I named | `mcp/docs/api/environment-transfer#delete-mode-removes-every-target-only-entity-of-a-kind` |
| Can an overwritten error page be restored after a transfer | `mcp/docs/api/environment-transfer#an-overwrite-keeps-the-old-version-except-on-error-pages` |
| Where does an administrator's theme or notification preference live | `mcp/docs/api/admins-and-permissions#personal-settings-live-on-the-admin-account` |
| Why is highContrast null on an admin account | `mcp/docs/api/admins-and-permissions#personal-settings-live-on-the-admin-account` |
| How do I set the high-contrast default for every administrator | `mcp/docs/api/settings#general-settings` |
| Which general setting can be read without a token | `mcp/docs/api/settings#one-general-setting-can-be-read-without-a-token` |
| Why does every public list stop at ten | `mcp/docs/api/content-api-permission-rules#a-restricted-read-caps-the-list-at-ten` |
| Why can a public token upload a file | `mcp/docs/api/content-api-permission-rules#why-a-public-token-can-upload-a-file` |
| Where is the restricted read length set | `mcp/docs/api/content-api-permission-rules#where-the-restricted-length-is-set-and-read` |
| Why is changing an AI account refused with 403 | `mcp/docs/api/ai-gateway#who-may-create-change-or-delete-an-account` |
| Why is every AI job refused with 429 | `mcp/docs/api/ai-gateway#refused-with-429-because-the-account-limit-is-used-up` |
| Why did an AI job complete with an empty result | `mcp/docs/api/ai-gateway#a-completed-job-with-an-empty-result-ran-out-of-budget` |
| Why can I not delete a screen template | `mcp/docs/api/ui-component-configs#a-template-that-is-still-assigned-cannot-be-deleted` |
| Why is saving the constructor agent's template refused with 400 | `mcp/docs/api/ui-component-configs#saving-a-template-that-refers-to-an-unsaved-recipe` |
| Why did assigning a template wipe the rest of the screen | `mcp/docs/api/ui-component-configs#a-record-page-template-covers-one-part-of-the-page` |
| Why is my field layout for a record page not applied | `mcp/docs/api/ui-component-configs#the-attribute-set-is-fixed-when-the-template-is-saved` |
| Which key opens the public API error listing | `mcp/docs/api/content-api-journal#two-listings-and-two-separate-permission-keys` |
| Why does clearing collection rows answer 403 | `mcp/docs/api/collections#deleting-rows-needs-the-row-key-not-the-collection-key` |
| Which key opens the application token listing | `mcp/docs/api/settings#treat-the-token-listing-as-a-secret` |
| Why does updating an admin answer 400 about unfilled fields | `mcp/docs/api/admins-and-permissions#an-admin-update-needs-login-and-email-beside-permissions` |
| Which key is a record page or a section tab panel saved under | `mcp/docs/api/ui-component-configs#screen-keys-a-record-page-is-saved-under` |

## Answers about writes that changed nothing

| Question | Go to |
|---|---|
| Why is what I just wrote missing from a list or a public read | `mcp/docs/api/index-attributes#when-a-written-value-becomes-searchable` |
| How do I know when a value is public | `mcp/docs/api/index-attributes#how-to-tell-when-a-value-is-public` |
| How far behind is indexing right now | `mcp/docs/api/index-attributes#how-far-behind-the-instance-is-right-now` |
| Do unmoving queue counters mean indexing has stopped | `mcp/docs/api/index-attributes#the-counters-cover-one-stage-and-not-the-whole-of-indexing` |
| Why does creating this fail as a duplicate | `mcp/docs/api/baseline-data#silent-duplicates-versus-hard-failures` |
| Why did my write answer 200 and change nothing | `mcp/docs/api/silent-no-ops` |
| Why did my menu item jump to the top level after an update | `mcp/docs/server/payload-conventions#an-omitted-field-can-mean-clear-it` |
| Why does a re-run of my import duplicate everything | `mcp/docs/api/import#re-running-an-import` |
| Why did my block disappear from its pages after an edit | `mcp/docs/api/blocks#what-an-update-does-to-the-pages-a-block-is-on` |
| Why did submissions vanish after a form edit | `mcp/docs/api/forms-and-form-data#an-update-drops-every-config-you-do-not-send-back` |
| How do I delete something safely | `mcp/docs/server/confirm-and-dry-run` |
| Why did my page update answer 200 with a body of false | `mcp/docs/api/pages#update-a-page` |
| Why is my journal time filter being ignored | `mcp/docs/api/journal#reading-the-journal` |
| Why did deleting a menu item leave its children behind | `mcp/docs/api/menus#delete-a-menu-or-an-item` |
| Why did deleting a parent page not delete its children | `mcp/docs/api/pages#delete-a-page` |
| Why did my block attachment by marker fail | `mcp/docs/api/pages#attaching-blocks` |
| Why is my block still nested after I sent blockPages | `mcp/docs/api/blocks#a-bare-page-id-in-blockpages-keeps-the-nesting` |
| Why does clearing the journal answer 400 | `mcp/docs/api/journal#clearing-the-journal-needs-an-explicit-window` |
| Why is the session traffic listing empty | `mcp/docs/api/journal#reading-the-session-traffic-listing` |
| Why is my public API error window being ignored | `mcp/docs/api/content-api-journal#this-listing-takes-a-flat-window-not-a-nested-one` |
| Why did deleting a few collection rows empty the collection | `mcp/docs/api/collections#deleteall-overrides-the-ids-you-send-with-it` |
| Why does a refund preview answer an amount of zero | `mcp/docs/api/orders#what-a-refund-body-ignores-without-saying-so` |
| Why did my admin token stop working after another sign-in | `mcp/docs/api/journal#what-closes-a-session-record` |

## Answers about search and filters

| Question | Go to |
|---|---|
| Why does my filter return nothing | `mcp/docs/api/filters#only-indexed-attributes-can-be-filtered` |
| Why does a page listing by ids return nothing | `mcp/docs/api/page-listing-filters#a-listing-without-parentid-reads-the-top-level-only` |
| Why does a listing answer 400 for an empty filter | `mcp/docs/api/interface-module-data#an-empty-filter-answers-400` |
| How do I pin a value in a facet | `mcp/docs/api/filters#pinning-is-set-on-an-attribute-not-on-a-value` |
| Why did a new value appear in my pinned facet by itself | `mcp/docs/api/filters#a-pinned-attribute-picks-up-new-values-on-its-own` |
| Why does a search by meaning return nothing for a kind | `mcp/docs/api/index-attributes#why-a-search-by-meaning-finds-nothing-for-a-kind` |
| Why does the site search return only five results | `mcp/docs/api/global-search#results-come-back-grouped-in-fives` |
| Why is limit refused on the search | `mcp/docs/api/global-search#paging-one-kind-needs-limit-and-exactly-one-type` |
| Why does the search never find my documents | `mcp/docs/api/global-search#what-global-search-never-finds` |
| How do I search inside an attached document | `mcp/docs/api/file-content-search#a-result-is-a-whole-file-and-not-a-fragment` |
| Why does my document search return an empty list | `mcp/docs/api/file-content-search#an-empty-result-that-has-a-reason-says-so` |
| How do I show "did you mean" on a site search | `mcp/docs/api/search-query-correction#correct-a-query-directly` |
| Why did the public search answer for different words | `mcp/docs/api/search-query-correction#correction-inside-the-public-searches` |
| Where do I tune or switch off typo correction | `mcp/docs/api/search-query-correction#tuning-correction-for-a-project` |
| Why did my typo-correction settings write change nothing | `mcp/docs/api/search-query-correction#tuning-correction-for-a-project` |
| How do I keep one attribute out of typo correction | `mcp/docs/api/attribute-sets#changing-an-attribute-afterwards` |
| How do I rebuild the word list behind typo correction | `mcp/docs/api/search-query-correction#is-the-word-list-up-to-date` |
| Where is the public search by meaning | `mcp/docs/api/global-search#semantic-search-over-one-kind-of-entity` |
| Which kinds may I pass in `types` | `mcp/docs/api/global-search#which-kinds-the-admin-side-accepts` |
| Can the search find a payment or AI account, a filter or a module | `mcp/docs/api/global-search#settings-records-the-admin-search-also-finds` |
| Can a stored credential be found through the search | `mcp/docs/api/global-search#settings-records-the-admin-search-also-finds` |

## Answers about attributes and values

| Question | Go to |
|---|---|
| Why did my attributes come back empty | `mcp/docs/api/attribute-sets#two-levels-always` |
| Does an empty `attributeValues` mean no values or not published yet | `mcp/docs/api/index-attributes#what-issync-tells-you-and-what-attributevalues-does-not` |
| How long do I keep re-reading while `isSync` is false | `mcp/docs/api/index-attributes#an-unpublished-product-can-be-missing-rather-than-empty` |
| How do I read one attribute value without the set | `mcp/docs/api/attribute-sets#reading-one-value-by-its-attribute-marker` |
| Why are my validators empty when a site reads the form | `mcp/docs/api/attribute-sets#two-reads-two-answers` |
| How do I put an image or a colour on a list option | `mcp/docs/api/list-options-and-extra-values#extra-values-sit-in-extended-with-no-locale-key` |
| Why does the panel show one selected option out of four | `mcp/docs/api/list-options-and-extra-values#several-selected-options-need-multiselect` |
| Where do I put button captions and interface text | `mcp/docs/api/content-modelling#where-interface-labels-belong` |
| How do I write a list or radioButton value | `mcp/docs/api/attribute-types#the-value-form-of-a-list-and-a-radiobutton` |
| How do I copy attribute values into another locale | `mcp/docs/api/attribute-sets#copying-values-from-one-locale-into-another` |
| Why did my value copy answer queued instead of a count | `mcp/docs/api/attribute-sets#a-copy-can-finish-in-the-call-or-carry-on-after-it` |
| What belongs in a page's config object | `mcp/docs/api/pages#the-config-object-is-a-free-map-of-numbers` |

## Answers about content structure and ordering

| Question | Go to |
|---|---|
| Where is the product list operation and why does it want `langCode` | `mcp/docs/api/products#listing-products` |
| How do I create a menu with its pages | `mcp/docs/api/menus#creating-a-menu-with-its-pages` |
| Why did my menu arrive with more items than it has | `mcp/docs/api/menus#parent-references-are-polymorphic` |
| Why does a hidden page still show in my menu | `mcp/docs/api/menus#why-a-hidden-page-still-shows-in-a-public-menu` |
| How do I set the order of menu items | `mcp/docs/api/menus#ordering-items` |
| How do I order products on a page and why did my reorder do nothing | `mcp/docs/api/pages#ordering-products-on-a-page` |
| Why is my product price still the old number | `mcp/docs/api/products#prices-and-the-price-attribute` |

## Answers about files and previews

| Question | Go to |
|---|---|
| Why do my uploaded images have no preview | `mcp/docs/api/files-and-uploads#no-preview-template-no-preview-and-no-error` |
| How do I upload a file through this server | `mcp/docs/server/cms-upload-file` |
| Where does alt text for an image live | `mcp/docs/api/files-and-uploads#a-file-record-has-nowhere-to-keep-alt-text` |
| Why is a document I attached not searchable | `mcp/docs/api/file-content-search#why-a-document-you-attached-is-not-found` |
| Why do old images still show the old thumbnail size | `mcp/docs/api/templates-and-previews#changing-a-preview-template-does-not-re-crop-old-images` |
| I regenerated previews and one image is unchanged | `mcp/docs/api/templates-and-previews#one-unchanged-preview-does-not-mean-it-failed` |

## Answers about forms and reviews

| Question | Go to |
|---|---|
| What do I send as the value of a captcha field | `mcp/docs/api/captcha-fields#what-a-captcha-field-must-carry` |
| Why is my captcha field accepting any token | `mcp/docs/api/captcha-fields#hiding-the-captcha-field-switches-the-check-off` |
| Where do the rating and aggregation settings live | `mcp/docs/api/rating-forms-and-reviews#the-rating-settings-live-in-the-module-config` |
| How do I import reviews from another system | `mcp/docs/api/rating-forms-and-reviews#importing-reviews-from-another-system` |
| What is averageValue next to the star rating | `mcp/docs/api/rating-forms-and-reviews#what-the-rating-aggregate-carries` |
| Why was my second review refused as already rated | `mcp/docs/api/rating-forms-and-reviews#only-a-submission-with-a-score-counts-as-a-rating` |
| Why is my captcha refusing a token that looks right | `mcp/docs/api/captcha-fields#a-valid-token-can-still-be-refused` |
| Why does my submission date filter answer 400 | `mcp/docs/api/form-submissions#how-to-write-a-date-bound-the-filter-accepts` |
| How do I list the submissions of one form | `mcp/docs/api/form-submissions#the-listing-is-a-post-with-the-filter-in-the-body` |
| Why does creating or editing a submission answer 403 | `mcp/docs/api/form-submissions#creating-or-editing-a-submission-needs-a-write-key` |
| Why is formModuleConfigId refused by the filter body | `mcp/docs/api/form-submissions#what-formmoduleconfigid-narrows-and-what-it-cannot` |
| Why does a submitted field come back as null | `mcp/docs/api/sensitive-form-fields#why-a-submitted-value-comes-back-as-null` |
| Who can read a sensitive form field | `mcp/docs/api/sensitive-form-fields#who-can-read-the-value` |

## Answers about events and notifications

| Question | Go to |
|---|---|
| Why is my event refused on this module | `mcp/docs/api/events#an-event-on-another-module-is-refused` |
| How do I create a notification template a workflow can call | `mcp/docs/api/events#an-event-on-another-module-is-refused` |
| Why does my form event send no mail | `mcp/docs/api/events#an-event-with-no-recipient-looks-configured` |
| Did my event actually send anything | `mcp/docs/api/events#checking-whether-an-event-actually-sent-mail` |
| Why did my event send a mail but no push | `mcp/docs/api/events#which-channels-an-event-sends-on` |

## Answers about visitors and sessions

| Question | Go to |
|---|---|
| How does a visitor sign out | `mcp/docs/api/content-api-sign-in-and-cart#where-the-session-routes-live` |
| How does a visitor reset a forgotten password | `mcp/docs/api/password-reset#two-calls-in-order` |
| Why is the password change asking for a code | `mcp/docs/api/password-reset#the-code-is-required-on-every-provider` |
| Why is my guest cart empty after adding to it | `mcp/docs/api/content-api-sign-in-and-cart#the-guest-cart-needs-a-uuid-in-x-guest-id` |
| How do I turn a login into a user id | `mcp/docs/api/users-and-groups#find-a-user-by-their-login` |
| How do I renew an administrator token | `mcp/docs/api/admin-sign-in#renewing-an-administrator-token` |
| Why did an admin lose its session after a permission edit | `mcp/docs/api/admins-and-permissions#a-permission-change-signs-the-target-admin-out` |
| Why can a sign-in provider not be deleted | `mcp/docs/api/users-and-groups#a-sign-in-provider-with-users-cannot-be-deleted` |
| How do I issue an application token | `mcp/docs/api/settings#issuing-an-application-token` |
| How do I delete a key from a visitor state object | `mcp/docs/api/content-api-sign-in-and-cart#removing-a-key-from-the-visitor-state-object` |

## Answers about commerce and product blocks

| Question | Go to |
|---|---|
| Why is my coupon working for everybody | `mcp/docs/api/discounts#which-operation-created-a-coupon-decides-its-reuse` |
| Why does my similar-products block answer 400 | `mcp/docs/api/similar-product-blocks#manual-or-automatic-decides-what-a-request-returns` |
| Why is my similar-products block empty or short | `mcp/docs/api/similar-product-blocks#why-a-block-comes-back-short` |
| Why do two order statuses both say they are the default | `mcp/docs/api/order-statuses#which-axis-a-status-belongs-to` |
| Why did my new order status land on the payment axis | `mcp/docs/api/order-statuses#which-axis-a-status-belongs-to` |
| Where is the operation that transitions an order status | `mcp/docs/api/order-status-triggers#there-is-no-such-thing-as-a-transition` |
| Why did change-status answer 200 without moving the payment status | `mcp/docs/api/order-statuses#the-payment-axis-is-managed-for-you` |
| How do I add a status trigger of my own | `mcp/docs/api/order-status-triggers#a-storage-can-define-triggers-of-its-own` |
| Why is my subscription status never equal to cancelled or expired | `mcp/docs/api/subscriptions#plans-and-enrolments` |
| Why can I not delete a subscription plan | `mcp/docs/api/subscriptions#delete-a-plan` |
| How is the body of a refund shaped | `mcp/docs/api/orders#previewing-a-refund-before-issuing-one` |
| Why does a refund answer 400 when its preview worked | `mcp/docs/api/orders#why-a-refund-answers-400-when-its-preview-worked` |
