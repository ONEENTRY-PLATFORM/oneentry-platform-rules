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
| `mcp/docs/api/global-search` | Locating an entity by a word a human used |

## Commerce

| docId | Read it when |
|---|---|
| `mcp/docs/api/orders` | Reading or changing orders |
| `mcp/docs/api/order-statuses` | Moving an order between states |
| `mcp/docs/api/payments` | Payment accounts, sessions or refunds |
| `mcp/docs/api/discounts` | Discounts, coupons and how they combine |
| `mcp/docs/api/subscriptions` | Recurring plans and customer enrolments |

## Forms and people

| docId | Read it when |
|---|---|
| `mcp/docs/api/forms-and-form-data` | Defining a form, or binding it so it accepts anything |
| `mcp/docs/api/form-submissions` | Listing, filtering or reading what visitors submitted |
| `mcp/docs/api/rating-forms-and-reviews` | Reviews, star ratings, or importing either |
| `mcp/docs/api/users-and-groups` | A Content API route returns a permission error |
| `mcp/docs/api/content-api-sign-in-and-cart` | Building the account or cart pages of a site |
| `mcp/docs/api/admins-and-permissions` | An operation was refused for lack of a permission |

## Instance configuration

| docId | Read it when |
|---|---|
| `mcp/docs/api/modules` | Explaining why an admin section is missing |
| `mcp/docs/api/events` | Sending a mail or a push when something happens |
| `mcp/docs/api/settings` | Instance-wide settings, limits and quota |
| `mcp/docs/api/import` | Bulk-loading content from a file |
| `mcp/docs/api/ai-gateway` | The platform's own model-calling feature |

## Answers to the questions asked most

| Question | Go to |
|---|---|
| Why did my attributes come back empty | `mcp/docs/api/attribute-sets#two-levels-always` |
| Why is what I just wrote missing from a list or a public read | `mcp/docs/api/index-attributes#when-a-written-value-becomes-searchable` |
| Why was my call refused with nothing sent | `mcp/docs/server/allow-levels#a-level-refusal-happens-before-authentication` |
| Why does my filter return nothing | `mcp/docs/api/filters#only-indexed-attributes-can-be-filtered` |
| Why does creating this fail as a duplicate | `mcp/docs/api/baseline-data#silent-duplicates-versus-hard-failures` |
| Where is the product list operation | `mcp/docs/api/products#listing-products` |
| Why does the product list complain about `langCode` | `mcp/docs/api/products#listing-products` |
| Why did my write answer 200 and change nothing | `mcp/docs/api/silent-no-ops` |
| Why are my validators empty when a site reads the form | `mcp/docs/api/attribute-sets#two-reads-two-answers` |
| How do I write a text field in a submission | `mcp/docs/api/attribute-types` |
| Why do my uploaded images have no preview | `mcp/docs/api/files-and-uploads#no-preview-template-no-preview-and-no-error` |
| Why did my menu item jump to the top level after an update | `mcp/docs/server/payload-conventions#an-omitted-field-can-mean-clear-it` |
| How do I create a menu with its pages | `mcp/docs/api/menus#creating-a-menu-with-its-pages` |
| How do I upload a file through this server | `mcp/docs/server/cms-upload-file` |
| Where do the rating and aggregation settings live | `mcp/docs/api/rating-forms-and-reviews#the-rating-settings-live-in-the-module-config` |
| How do I put an image or a colour on a list option | `mcp/docs/api/list-options-and-extra-values#extra-values-sit-in-extended-with-no-locale-key` |
| Why does the panel show one selected option out of four | `mcp/docs/api/list-options-and-extra-values#several-selected-options-need-multiselect` |
| Why did my menu arrive with more items than it has | `mcp/docs/api/menus#parent-references-are-polymorphic` |
| How do I set the order of menu items | `mcp/docs/api/menus#ordering-items` |
| Why is my event refused on this module | `mcp/docs/api/events#an-event-on-another-module-is-refused` |
| Why does my form event send no mail | `mcp/docs/api/events#an-event-with-no-recipient-looks-configured` |
| How do I write a list or radioButton value | `mcp/docs/api/attribute-types#the-value-form-of-a-list-and-a-radiobutton` |
| Why is my coupon working for everybody | `mcp/docs/api/discounts#which-operation-created-a-coupon-decides-its-reuse` |
| How do I import reviews from another system | `mcp/docs/api/rating-forms-and-reviews#importing-reviews-from-another-system` |
| Where does alt text for an image live | `mcp/docs/api/files-and-uploads#a-file-record-has-nowhere-to-keep-alt-text` |
| Why do old images still show the old thumbnail size | `mcp/docs/api/templates-and-previews#changing-a-preview-template-does-not-re-crop-old-images` |
| I regenerated previews and one image is unchanged | `mcp/docs/api/templates-and-previews#one-unchanged-preview-does-not-mean-it-failed` |
| Did my event actually send anything | `mcp/docs/api/events#checking-whether-an-event-actually-sent-mail` |
| Why did my block disappear from its pages after an edit | `mcp/docs/api/blocks#what-an-update-does-to-the-pages-a-block-is-on` |
| Why does my similar-products block answer 400 | `mcp/docs/api/similar-product-blocks#manual-or-automatic-decides-what-a-request-returns` |
| Why is my similar-products block empty or short | `mcp/docs/api/similar-product-blocks#why-a-block-comes-back-short` |
| Why does every public read answer 403 | `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header` |
| Why is my guest cart empty after adding to it | `mcp/docs/api/content-api-sign-in-and-cart#the-guest-cart-needs-a-uuid-in-x-guest-id` |
| Why did submissions vanish after a form edit | `mcp/docs/api/forms-and-form-data#an-update-drops-every-config-you-do-not-send-back` |
| How do I delete something safely | `mcp/docs/server/confirm-and-dry-run` |
