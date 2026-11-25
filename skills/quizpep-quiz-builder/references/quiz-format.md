# Quizpep quiz document format (schemaVersion 1)

Authoritative source: `resources/schemas/quiz.schema.json` (JSON Schema 2020-12, 35 `$defs`) plus the
semantic checks in `App\Quiz\QuizValidator`. Use `get_quiz_format` with a `$defs` name (e.g.
`ResultSection`, `Settings`, `ProductCard`) for the exact sub-schema of anything summarised here.

Golden rules
- Every fixed-shape object has `additionalProperties: false`: an unknown or misspelled key is an error.
- **Never send an empty object `{}`** for an optional object (`style`, `card`, `discount`, `couples`,
  `layout`…). The server decodes `{}` as an empty PHP array and the schema rejects it ("must match
  the type: object"). Omit the key instead. (`intro`, `design`, `ui`, `settings.fallback`,
  `settings.addToCart`, `settings.layout`, `settings.resultsEmail` are auto-dropped when empty; nothing else is.)
- Omitted optional fields take the defaults listed below. Don't write defaults out unless they matter.
- `validate_quiz` before `create_quiz` / `update_quiz`. Errors block saving/publishing; warnings don't.

---

## 1. Top level

| Key | Req | Type | Notes |
|---|---|---|---|
| `schemaVersion` | yes | `1` | const |
| `title` | yes | LocalizedText | quiz name (also the results heading fallback) |
| `defaultLocale` | yes | Locale | must also appear in `locales` |
| `locales` | yes | Locale[] | ≥1, unique. Locale = `en` or `en-GB` (`^[a-z]{2}(-[A-Z]{2})?$`) |
| `settings` | yes | Settings | `scope` and `resultCount` required |
| `intro` | no | Intro | omit → quiz starts at question 1 |
| `questions` | yes | Question[] | ≥1 |
| `results` | yes | Results | `layout` required |
| `design` | no | Design | |
| `ui` | no | UiTexts | button/helper text overrides |

## 2. Ids and references

- Id pattern: `^[A-Za-z0-9_-]{1,64}$` (no spaces or dots). Use short snake_case: `skin_type`, `oily`.
- Must be unique (error otherwise):
  - question ids across the quiz; form field ids across the quiz **and** distinct from question ids;
  - answer ids within one question (the same answer id may repeat in different questions);
  - manual result ids; section ids within one results page; personality-category ids (couples).
- References that must exist (error otherwise): `next.questionId` → a question (and not the question
  itself); `PathCondition.questionId` → a question, `answerIds` → answers of that question;
  `personality` signal `personalityId` → `results.personalities[].id`; `question.category` →
  `settings.couples.categories[].id`.
- Personality ids are only checked for existence when `results.layout = "personality"`.
- Keep ids stable after publishing: analytics, saved answers, manual paths and `{{id}}` piping
  refer to them. Change labels, not ids.

## 3. Text

**LocalizedText**: `{"en": "Skin type?", "fr": "Type de peau ?"}`, each value ≤ 2000 chars
(LongLocalizedText ≤ 10000: personality `description`, `footnote`, couples `prompts`, email `disclaimer`).
- The default-locale entry must exist and be non-blank (error). Missing other locales → warning
  (default text is shown).

**Rich text (HTML subset)**. Allowed tags: `b strong i em u s a br span code` and, outside headings,
`p div ul ol li`. Links: `href` must start with `https:`, `http:`, `mailto:`, `tel:`, `/` or `#`;
external links open in a new tab. Everything else (scripts, styles, attributes) is stripped, text kept.

| Field | Renders as |
|---|---|
| `intro.heading`, section `heading`/`imageHeading`, custom block `heading` | inline rich (b/i/u/a/br…) |
| `intro.body`, `results.body`, manual result `body`, personality `description`, question `footnote`, section `body`/`imageBody`, FAQ `answer`, block `body`, couples category `prompts` | full rich text |
| question `title`/`description`, answer `label`, `results.heading`, FAQ `question`, button labels, `ui.*` | plain text (tags show literally) |
| `ResultSection.html` (type `html`) | raw HTML/CSS, not sanitised, ≤ 20000 chars |

Plain text with no tags keeps its line breaks.

**Piping**: `{{id}}` or `{{id|fallback}}` inserts an earlier answer (answer labels, number + unit,
typed text; form fields by field id). Works in question `title`/`description`, `results.heading`/`body`,
manual result heading/body and `settings.resultsEmail` texts. A token for a question that isn't on an
earlier page → warning. Dangling punctuation is tidied ("Hi {{name}}," → "Hi").

---

## 4. Settings

| Key | Req | Type / values | Default | Notes |
|---|---|---|---|---|
| `scope.type` | yes | `all` \| `collections` | | products the quiz may recommend |
| `scope.collectionIds` | | string[] | | collection **ids** (`gid://shopify/Collection/123` or `123`) |
| `resultCount` | yes | int 1–24 | | products shown (grid/personality) |
| `hideOutOfStock` | | bool | true | |
| `tieBreak` | | `best_selling` `newest` `price_asc` `price_desc` `inventory` | best_selling | equal scores; best_selling currently = most stock |
| `maxPerProductType` | | int ≥1 | | cap per product type in the results |
| `fallback.collectionId` | | string | | collection **id** (not handle) |
| `fallback.productIds` | | string[] | | product ids. No fallback → warning |
| `autoAdvance` | | bool | true | single-select goes on as soon as picked |
| `redirectImmediately` | | bool | false | redirect answers fire even when autoAdvance is off |
| `addToCart` | | `{enabled, showAddAll, afterAdd: stay\|cart, afterAddAll, tagLineItems, uniqueOnly}` | enabled, showAddAll, stay | |
| `discount` | | `{code (req, 1–64), message, placement: results\|before_results\|both, requireEmail, autoApply}` | results, false, true | create the code in Shopify Discounts; quiz doesn't create it |
| `resultsEmail` | | see below | | |
| `notifications` | | `{email, submissions}` | | email the merchant each completed quiz |
| `tagCustomers` | | bool | true | Shopify customers: tag with answers' `customerTag` |
| `recommendations` | | `{quantities, plainPrices}` | false | `quantities` enables outcome `quantities` |
| `layout` | | `{minHeight 0–1200, animation, pageAnimation: fade\|slide\|none, autoScroll: never\|mobile\|always, scrollOffset 0–300}` | | |
| `tracking` | | `{meta, google, urlParams, metaEvents{…}, googleEvents{…}}` | true, true, false | event-name overrides ≤60 chars |
| `integrations` | | `{properties{quizName, completedAt, discountCode, products, answers}, lists{<app>: {id, name}}}` | | |
| `couples` | | see §13 | | |

`discount.requireEmail` without any email/phone question → warning (nobody sees it).

**resultsEmail** (`enabled` required). Sent to every shopper who gave an email (email question or form
email field), whether or not they ticked marketing consent. Enabled with no email question → warning.
Keys: `subject`, `heading`, `body`, `buttonLabel`, `productsTitle`, `answersTitle`, `retakeTitle`,
`retakeButtonLabel`, `discountTitle`, `discountBody`, `discountButtonLabel` (LocalizedText, piping ok);
`disclaimer` (Long); `showDiscount` (true), `showAnswers` (false), `showRetake` (true), `showResult`
(true, personality), `uppercaseButtons` (false); `imageUrl` (logo), `logoSize` small|medium|large,
`discountButtonUrl`.

---

## 5. Intro (optional)

`heading`, `body` (rich), `buttonLabel`, `imageUrl`, `mobileImageUrl`, `imageAlt`,
`layout`: `stack` | `split_right` | `split_left` (image fills half), `imageAlign`, `imageWidth` 40–1200,
`imageRadius`, `imageFocusX/Y` 0–100, `headingSize` small|medium|large|xlarge, `headingColor`,
`subheadingSize`, `subheadingColor` (`#rrggbb`), `buttonStyle` primary|outline|link, `buttonColor`
primary|secondary|tertiary, `buttonSize`, `buttonFullWidth`, `buttonArrow`, `style` (SectionStyle),
`hidden` (skip intro, keep content), `blocks`: ordered `[{type: heading|subheading|image|button, hidden}]`
(default heading, subheading, image, button).

---

## 6. Questions

Common fields (`id`, `type`, `title` required):

| Key | Type | Notes |
|---|---|---|
| `description` | LocalizedText | plain text under the title |
| `stepLabel` | LocalizedText | a one- or two-word name ("Skin type") shown by the `tabs`, `side_list` and `numbered` progress styles |
| `required` | bool, default true | false → shopper can skip |
| `propertyName` | `^[a-z][a-z0-9_]{0,39}$` | key used when syncing the answer to Klaviyo/metafields |
| `answers` | Answer[] | choice types only (ignored + warning on others) |
| `next` | NextStep | question-level jump (§8) |
| `nextLabel` | LocalizedText | this page's Next button text |
| `footnote` | LongLocalizedText | rich text under the buttons |
| `media` | PageMedia | `{imageUrl, mobileImageUrl, alt, layout: right\|left\|background, overlay 0–90, overlayColor, mobile: top\|hidden, focusX, focusY}` |
| `style` | SectionStyle | page look |
| `nextButton`, `backButton` | ButtonLook | `{color: primary\|secondary\|tertiary, style: primary\|outline\|link, size, fullWidth}` |
| `shuffleAnswers` | bool | random answer order |
| `selectionTray` | bool | multiple choice with `maxSelections` ≤ 8: a tray of that many slots under the answers fills as the shopper picks (build-your-bundle) |
| `category`, `weigh` | Id, bool | couples quizzes only (§13) |

### Types

| `type` | Answer value | Type-specific fields | Rules |
|---|---|---|---|
| `single_choice` | one answer id | `answers` | ≥2 answers |
| `multiple_choice` | answer id[] | `answers`, `minSelections` ≥0, `maxSelections` ≥1 | ≥2 answers; min ≤ max |
| `image_choice` | id, or id[] if `maxSelections` > 1 | `answers` with `imageUrl`, `imageLabels`: below\|hidden | ≥2 answers |
| `dropdown` | one answer id | `answers` | ≥2 answers; native select |
| `slider` | number | `range` **required**, `rangeRule` | min < max |
| `number` | number | optional `range`, `rangeRule` | |
| `text` | string | | free text |
| `email_capture` | string (email) | `consent` | validated as an email |
| `form` | each field by its own id | `fields` (1–8), `consent` if a field is `email` | |
| `statement` | none | | info page with a Next button, no input |
| `file_upload` | uploaded file id | `upload: {accept: images\|images_pdf, maxSizeMb 1–20}` | |

- `range`: `{min, max, step (>0), unit}`; `unit` starting with a currency sign shows before the number
  (`$55`), otherwise after (`30 ml`).
- `rangeRule` (slider/number only, else error): `{attribute: {source: price|metafield, key}, operator:
  lte|gte|closest, mode: filter|score}` (mode default `filter`). See §9.
- `consent` (`email_capture`, or `form` with an email field; else error): `{enabled (req), label}`. Shows
  a marketing checkbox; only shoppers who tick it are marked as subscribed.
- `maxSelections` on non-multi types → warning.

### FormField (`form.fields[]`; `id`, `type`, `label` required)

`type`: `text` | `email` | `phone` | `number` | `date` | `dropdown`; `placeholder`; `required` (default
**false**, unlike questions); `propertyName`; `options` (LocalizedText[], ≥2 for dropdown, error).
Only the first email field is the shopper's email (>1 → warning). A phone field also unlocks
`discount.requireEmail`. Field ids are pipeable: `{{first_name}}`.

---

## 7. Answers (choice questions)

| Key | Req | Notes |
|---|---|---|
| `id` | yes | unique within the question |
| `label` | yes | plain LocalizedText |
| `imageUrl` | | absolute URI; shown on any choice button, main use `image_choice` |
| `icon` | | an icon name, e.g. `coffee`, `moon-stars`, `paw-print` (`get_quiz_format` part `Icons` lists them by topic). Shown on text answers; answers with `imageUrl` and `image_choice` show the picture instead. Unknown name → warning, no icon |
| `description` | | plain LocalizedText, a short second line under the label (hidden in `pills`) |
| `signals` | | Signal[] (§9) |
| `customerTag` | | ≤255 chars, Shopify customer tag added when chosen (`tagCustomers`) |
| `next` | | NextStep (§8), single-select only |
| `redirectUrl` | | absolute URI; choosing it sends the shopper there, skipping results |
| `exclusive` | | multiple choice "None of these": clears the others |
| `unknown` | | couples: "Unsure" answer, counts as neither agreement nor difference |

---

## 8. Navigation (branching, skips, jumps)

`NextStep = {"to": "question", "questionId": "<id>"}` or `{"to": "results"}`.
- Order of precedence after a page: chosen answer's `next` → question's `next` → next question in array
  order → results after the last question.
- Answer `next` only applies on single-select pages (`single_choice`, `dropdown`, single `image_choice`).
  On `multiple_choice` the widget ignores answer `next`; use the question's `next`.
- Jumps may go forward or backward. The validator does **not** detect loops: never jump back to an
  earlier question that leads here again.
- Skipped questions are simply absent from the answers; their signals don't apply.
- Put branch-only questions after the main path and end each branch with `next: {to: results}` (or
  place it last) so one branch doesn't fall through into the other.
- `redirectUrl` on an answer ends the quiz immediately (no results, the run isn't completed).

---

## 9. Signals and product matching

A signal is one of six objects, distinguished by `kind`:

| `kind` | Fields | Effect |
|---|---|---|
| `prefer` | `attribute`, `values` (≥1), `match` any\|all (any), `weight` 1–5 (3) | +weight to matching products |
| `require` | `attribute`, `values`, `match` | hard filter: only matching products stay |
| `avoid` | `attribute`, `values` | hard filter: matching products removed |
| `pin` | `productId`, `variantId`, `weight` 1–10 (10) | +weight×2 to that product (beats any prefers) |
| `variant_hint` | `option`, `value` | pre-selects a variant on result cards, e.g. `Size`=`M` |
| `personality` | `personalityId`, `points` 1–10 (1) | points to a personality (layout personality only; else warning) |

**Attribute** `{source, key?}`:

| `source` | Matches | `key` |
|---|---|---|
| `tag` | product tags | – |
| `product_type` | product type | – |
| `vendor` | vendor | – |
| `collection` | collection id, handle **or** title | – |
| `option` | values of the option named `key` (e.g. Size, Color) | required (error) |
| `metafield` | not synced yet: matches nothing | required |
| `enriched` | not synced yet: matches nothing | required |

- Matching is exact after normalising: case-insensitive, spaces/`-`/`_` equivalent (`"Oily Skin"` =
  `"oily-skin"` = `"oily_skin"`). No partial/substring matching: `"oily"` does not match `"oily-skin"`.
  List both when unsure (`["oily-skin", "oily"]`).
- `match: all` = the product must have every value; `any` = at least one.
- Use only values that exist in the store (`get_store` → tags, product types, vendors, collections).
- Product ids: `gid://shopify/Product/123` or `123` (digits are extracted).

**How the engine ranks** (`App\Engine\Recommender`):
1. Collect signals from every chosen answer (all questions, skipped ones excluded) and range rules from
   answered slider/number questions with a `rangeRule`.
2. Candidates = products in scope (and in stock if `hideOutOfStock`).
3. Filter: `avoid`, `require`, and `rangeRule` with `mode: filter` (`lte`: price ≤ value, `gte`: price
   ≥ value). Products without a value for the attribute fail a range filter.
4. Score: sum of matching `prefer` weights + pins (weight×2) + `mode: score` ranges
   (3 × closeness, where closeness = 1 − |price − value| / value; the operator is ignored in score
   mode). Sort by score, then `tieBreak`, then product id.
5. Take `resultCount` (respecting `maxPerProductType`), or fill slots (§10).
6. **Never empty** – if step 3 leaves nothing: retry keeping only `avoid` (fallbackLevel 1) → merchant
   `settings.fallback` products (2) → any in-scope product (3).

Consequences
- `prefer` only ranks; it never removes. Products with score 0 still fill remaining slots. To make
  an answer decisive use `require` (and accept relaxation to level 1 when nothing matches).
- `operator: closest` only does something with `mode: score`; with `mode: filter` it neither filters
  nor scores.
- `rangeRule.attribute.source` must be `price` to have an effect today (`metafield` filters everything
  out → relaxed fallback).
- Reasons ("Because you said …") are the labels of the answers whose signals matched, plus range
  answers the product fits; shown when `results.showReasons` (default true).

---

## 10. Results

| Key | Type | Notes |
|---|---|---|
| `layout` (req) | `grid` \| `slots` \| `personality` | |
| `heading`, `body` | LocalizedText | body is rich; piping allowed |
| `showReasons` | bool (true) | "Because you said …" on cards |
| `showRetake` | bool (true) | |
| `mode` | `automatic` \| `manual` (automatic) | manual → `manual[]` checked first |
| `default` | `{outcome}` | when no manual result matches (default `{type: automatic}`) |
| `manual` | ManualResult[] ≤100 | |
| `slots` | Slot[] | layout slots: ≥1 (error) |
| `personalities` | Personality[] | layout personality: ≥2 (error) |
| `personalityRules` | object | optional |
| `sections` | ResultSection[] ≤30 | default header, products, footer |
| `sectionStyle` | SectionStyle | shared by sections with `useGlobalStyle` |
| `card` | ProductCard | shared product-card look |

**grid**: top `resultCount` products.

**slots** (routine, e.g. Cleanse → Treat → Moisturise): `slots: [{id, label, attribute, values}]`.
Each slot gets the best-ranked unused product matching its attribute; `resultCount` is ignored;
unfilled slots are reported as `emptySlots`. `option`/`metafield`/`enriched` slots need a `key`.

**personality**: answers give `personality` points; the winner's `title` replaces the results heading
and its `description` (rich) replaces the body. Product choice:
- personality `signals` with `pin` → exactly those products in that order (answer prefers ignored);
  if all are out of stock, no products show.
- other personality signals (prefer/require/avoid…) are added to the answers' signals, then normal
  ranking. No signals → products ranked by answer signals only.
- Personality fields: `id`, `title` (req), `description`, `imageUrl` (shown above the results
  heading when that personality wins), `signals`, `onlyWhenChosen` (only eligible if the shopper picked one of its
  answers outside `ignoreQuestionIds`).
- Without `personalityRules`: most points wins; ties → listed first.
- `personalityRules`: `show` 1–3 (results shown, best first), `fairScoring` (rank by share of
  achievable points), `keyQuestionIds` + `minQuestions` (default 2: eligible only with a point from a
  key question or from ≥ minQuestions questions), `ignoreQuestionIds`, `ties`: `first` | `mixed` (show
  all tied, with `mixedHeading`/`mixedText`) | `ask` (shopper picks; `tieQuestion`), `showScores`,
  `labels` (≤3, label above each shown result), `pairText` (`{first}`/`{second}` placeholders, show ≥2).
- There is no per-personality results page: all personalities share `results.sections`; the header
  section shows the winner's title and description.

**manual mode** (`mode: "manual"`): `manual[]` is checked top to bottom; the first whose `path`
matches wins, else `default.outcome`. Manual mode with no manual results → warning; a manual result
with an empty path matches everyone (warning) and hides the ones below it.

ManualResult: `id`, `name` (merchant-only, 1–100 chars), `path`, `outcome` (all required); `heading`,
`body` (replace results heading/body), `hideAddAll`, `sections` (own results page; missing → shared).

PathCondition: `{questionId (req), answerIds, match: any|all, min, max}`.
- All conditions in a path must hold (AND). Within one condition `answerIds` are OR (`any`) or all
  required (`all`, for multiple choice). `min`/`max` (inclusive) for number/slider answers; min > max →
  error. A listed question with no answerIds/min/max accepts anything.
- Unknown question or answer ids → error.

**Outcome** (one of):

| `type` | Fields | Shows |
|---|---|---|
| `automatic` | – | engine ranking (§9) |
| `products` | `productIds` (req, ≤24, non-empty), `quantities` `{productId: 1–99}` | those products in order (out of stock skipped; none left → automatic) |
| `collection` | `collection` (req, **handle**) | best matches inside that collection, answer filters applied, relaxed if empty |
| `text` | – | heading and body only, no products |
| `redirect` | `url` (req, `https://…`) | sends the shopper to that page |

---

## 11. Result sections

`ResultSection`: `id`, `type` required. One page ≤30 sections; at most one each of `header`,
`products`, `footer` (error); a page without `products` → warning.

| `type` | Purpose | Required (validator) | Main fields |
|---|---|---|---|
| `header` | results heading, body, discount | – | `style`, `badge` (a short value in a big circle above the heading, takes `{{answer}}` tokens, e.g. `{{usual_size}}`), `badgeLabel` |
| `products` | the recommendations | – | `card` |
| `footer` | Retake + Add all to cart | – | `retakeButton`, `showTotal` (total price of the products, for bundles) |
| `text` | heading + text | – | `subheading`, `heading`, `body`, `buttonLabel` + `link` (button needs both) |
| `image` | one image | `imageUrl` | `alt`, `link`, `mobileImageUrl` |
| `hero` | banner with text over image | `imageUrl` or `heading` | `subheading`, `heading`, `body`, `buttonLabel`, `link`, `overlay`, `overlayColor`, `textPosition`, `textBackground` |
| `collection` | featured collection | `collection` (handle) | `heading`, `limit` 1–12 (4), `card` |
| `image_product` | image beside one product | `productId`, `imageUrl` | `imagePosition` left\|right, text fields as `text` |
| `image_text` | image beside text | `imageUrl` | text fields as `text`, `imagePosition`, `imageHeading`/`imageBody` (text over the image) |
| `newsletter` | email sign-up | – | `heading`, `subheading`, `body`, `placeholder`, `buttonLabel`, `successMessage`, `consent`, `consentText` |
| `articles` | blog posts | `blog` (handle) | `limit` |
| `faq` | accordion | `faqs` 1–20 `[{question, answer}]` | `heading`, `body` |
| `custom` | 1–2 blocks side by side | `blocks` 1–2 | block `type`: text\|image (`imageUrl` req)\|video (`videoUrl` req)\|product (`productId` req)\|collection (`collection` req)\|newsletter |
| `html` | merchant's own HTML/CSS | non-blank `html` | ≤20000 chars, not sanitised |
| `highlights` | small icon cards (care guide, brewing tips, spec meters) | `items` 1–8 `[{icon?, title, text?, level? 0–5}]` | `heading`, `body`; `level` draws a 5-mark meter |

Shared optional section fields: `hidden`, `style` (SectionStyle), `useGlobalStyle`, `buttonStyle`
primary|outline|link, `buttonColor` primary|secondary|tertiary, `buttonSize`.

`collection` sections skip products already recommended. `image_product` shows nothing if the product
is unavailable.

**SectionStyle** (also `intro.style`, `question.style`, `design.questionStyle`): `desktopHeight` /
`mobileHeight`: auto|fifth|third|half|two_thirds|three_quarters|full|px500|px650|px800|image|custom
(+ `desktopHeightPx`/`mobileHeightPx` 100–2000), `background`, `textColor` (`#rrggbb`),
`backgroundImage`, `overlay` 0–90, `gutters`, `padding` 0–120, `top`/`bottom` 0–200, `align`
left|center|right, `width` full|wide|narrow, `radius` 0–40, `headingSize`, `visibility`
all|desktop|mobile, `animation` none|fade|slide|zoom.

**ProductCard** (`results.card`, section `card`): `layout`/`mobileLayout` grid|carousel, `columns` 1–4
(3), `mobileColumns` 1–2 (1), `imageRatio` square|portrait|landscape|natural|custom (+ `imageAspect`
0.5–2), `imageFit` cover|contain, `image` variant|featured|second, `hoverImage`, `gallery`
none|thumbnails|slideshow, `variants` hidden|dropdown|buttons|quick_add|separate, `showQuantity`,
`showVendor`, `showType`, `showDescription` (+ `descriptionLength` short|full), `showSavings`,
`saleBadge`, `tagBadge`, `hidePrice`, `hideLinks`, `hideAddToCart`, `hideSoldOutVariants`, `variantTitle`, `newTab`,
`fullWidthButton` (true), `textAlign` left|center, `buttonLabel`, `button` (ButtonLook), `imageRadius`
0–40, `fonts.{title,vendor,type,description,price,variant}` = `{color, style normal|italic|uppercase,
weight light…bold, size 8–48, mobileSize, letterSpacing -2–10}`.

`tagBadge` = `{prefix (req, 1–40), label}`: a badge on the product photo from the product's first tag
that starts with `prefix`; the rest of the tag fills `{value}` in `label`. Example:
`{"prefix": "ages-", "label": {"en": "Ages {value}"}}` turns the tag `ages-3+` into "Ages 3+";
`{"prefix": "players-", "label": {"en": "{value} players"}}` turns `players-2-4` into "2-4 players".
Products without a matching tag show no badge, so check the catalog's tags with `search_products`
first. If a sale badge is also shown, it moves to the other corner.

Product images, titles and prices always come from the catalog; never put them in the document.

---

## 12. Design and UI texts

**Design** (all optional): `inheritTheme` (true), `primaryColor`/`primaryTextColor`,
`secondaryColor`/`secondaryTextColor`, `tertiaryColor`/`tertiaryTextColor`, `textColor`,
`backgroundColor`, `headingColor` (all `#rrggbb`, 6 hex digits only), `cornerRadius` 0–32,
`headingFont`/`bodyFont`: theme | playfair_display | cormorant_garamond | lora | libre_baskerville |
dm_serif_display | poppins | montserrat | dm_sans | inter | nunito | work_sans | jost | raleway |
space_grotesk, `answerLayout` list|grid|pills (list; `image_choice` always shows image tiles),
`answerIcons` `{style: line|fill|line_fill (line, solid once chosen; default), size: small|medium|large,
frame: none|circle}`, `progressStyle` bar|segments|dots|steps|ring|numbered|tabs|percent|counter|icons|
side_list|goal|roman|none (`tabs`/`side_list` show each question's `stepLabel`), `progressIcon` (icon
name for `icons`, default star, and `goal`, default gift), `progressCounter`, `buttonStyle` filled|outline,
`inputStyle` outline|filled, `pageWidth` contained (760px)|wide (1200px)|full, `nextButton`,
`backButton` (ButtonLook), `questionStyle` (SectionStyle for every question page), `customCss`
(≤20000 chars).

`customCss` is inserted as a plain `<style>` on the storefront page, **not** auto-scoped. Prefix every
selector with the widget root `.pq-root` so the theme isn't affected, e.g.
`.pq-root .pq-title { letter-spacing: .02em; }`. Useful classes: `.pq-card`, `.pq-title`, `.pq-subtitle`,
`.pq-text`, `.pq-answers`, `.pq-answer` (`[aria-pressed=true]` when chosen), `.pq-input`,
`.pq-progress-bar`, `.pq-products`, `.pq-product`, `.pq-hero`, `.pq-faq`. CSS variables on `.pq-root`:
`--pq-accent`, `--pq-radius`.

**UiTexts** (`ui`, all LocalizedText, missing → built-in default): `start`, `next`, `back`, `skip`,
`seeResults`, `finding`, `addToCart`, `addAll`, `retake`, `because`, `soldOut`, `closeMatch`,
`discountHeading`, `discountSubheading`, `discountHeadingResults`, `copyCode`, `copied`, `continue`,
`noOptions`, `required`, `invalidEmail`, `invalidPhone`, `invalidNumber`, `uploadButton`,
`uploadHintImages`, `uploadHintAll`, `uploading`, `uploadRemove`, `uploadTooLarge`, `uploadWrongType`,
`uploadFailed`, `viewProduct`, `viewCart`, `checkout`, `addedToCart`, `save`, `unavailable`, `readMore`,
`tieQuestion`. Only set them to change wording or translate.

---

## 13. Couples quizzes (`settings.couples`)

With `enabled: true` the widget runs a different flow: choose a mode → each partner answers privately
→ consent to share → comparison map by category (plus products if the engine recommends any).

| Key | Notes |
|---|---|
| `enabled` | bool (false) |
| `modes` | ≥1 of `solo` (answer + guess partner's answers), `invite` (private link), `together` (one device) |
| `predictions` | bool (true): solo mode also asks what the partner would answer and how they know |
| `inviteDays` | 1–60 (14): invite link lifetime |
| `categories` | `[{id, title (req), prompts (rich, "Explore together" conversation prompts)}]`, ids unique |
| `texts` | overrides of built-in couples texts by key (`^[a-zA-Z]{1,40}$` → LocalizedText), e.g. `mapTitle`, `modeTitle`, `soloTitle`, `inviteTitle`, `togetherTitle`, `consentText`; `{partner}`/`{name}` placeholders |

- Only choice questions are compared; set each one's `category` (must exist, error) and optionally
  `weigh: true` (asks importance + flexibility; essential + not-flexible differences are listed apart).
- Mark "Unsure / Prefer not to say" answers with `unknown: true`.
- Enabled with no choice question → warning.
- Partner names are asked by the couples screens; no form needed.

---

## 14. Validation summary

Errors (block saving/publishing): schema violations (unknown key, wrong type/enum, bad id/locale/
colour pattern, non-URI `imageUrl`/`redirectUrl`/`link`, out-of-range numbers, empty `{}` objects);
missing default-locale text; duplicate ids (§2); dangling references (§2); choice question with <2
answers; slider without `range`; `range.min ≥ max`; `rangeRule` on a non-numeric question; `consent`
on a non-email question; `form` with 0 or >8 fields; dropdown field with <2 options; `minSelections >
maxSelections`; jump to self / missing `questionId`; `option`/`metafield`/`enriched` without `key`;
slots layout without slots; personality layout with <2 personalities; `products` outcome without ids,
`collection` outcome without handle, `redirect` outcome without `https://` URL; manual path min > max;
section problems (§11).

Warnings (allowed): missing translations; no `settings.fallback`; answers on non-choice questions;
`maxSelections` on single-select; `upload` on non-upload; >1 email field; `{{id}}` not from an earlier
page; personality points in a non-personality layout; manual mode with no/empty-path results; page
without a products section; `requireEmail`/`resultsEmail` without an email question; couples without
choice questions.

URI fields (`imageUrl`, `mobileImageUrl`, `backgroundImage`, `link`, `redirectUrl`, `videoUrl`) must be
absolute (`https://…`); relative paths like `/collections/all` fail the `uri` format. Use full store URLs
(`https://<shop-domain>/collections/all`), and https only (http shows as mixed content). Rich-text
`<a href="/…">` links may be relative.

---

## 15. Testing (`test_quiz`)

`test_quiz` runs the saved quiz's engine with an **answers map**: a JSON object keyed by question id
(form pages: keyed by **field id**; the form's own id is not used).

| Question type | Value | Example |
|---|---|---|
| `single_choice`, `dropdown`, single `image_choice` | answer id (string) | `"skin_type": "oily"` |
| `multiple_choice`, multi `image_choice` | array of answer ids | `"concerns": ["acne", "aging"]` |
| `slider`, `number` | number | `"budget": 40` |
| `text`, `email_capture` | string | `"email": "sam@example.com"` |
| `form` | one entry per field id | `"first_name": "Sam", "email": "sam@example.com"` |
| `statement`, `file_upload` | omit | |

- Omit questions the shopper wouldn't see on that branch (skipped questions must be absent, or their
  signals count). Unknown keys are ignored. `null`, `""` and `[]` count as unanswered.
- A single string for a multi-select is accepted as one id; an array for single-select also works.
- Navigation (`next`) is not simulated: the map is taken as the final set of answers.

```json
{"skin_type": "oily", "concerns": ["acne", "fragrance_free"], "budget": 30}
```

What `test_quiz` returns, and what to check:
- `products[]`: `id` (gid), `title`, `score`, `reasons[]`, `available`, `slot`. Best first.
- `fallback_level`: 0 = filters held; 1 = `require`/range filters relaxed (nothing matched them);
  2 = merchant fallback; 3 = any product. Anything > 0 on a main path is a dead end to fix.
- `score` 0 / empty `reasons` = the product matched none of the answers (filler). `prefer` never
  removes products, so level 0 alone doesn't prove a match: most products on a main path should
  have score > 0.
- `outcome` (automatic|products|collection|text|redirect) and `result_id` (matched manual result, or null).
- `personality`: `shown` (winners, best first), `tie_choice` (ties = ask: the shopper must choose
  first; pass their pick as `prefer` to test it), `mixed`, `points`.
- `section_products` (products for "Featured collection" sections, by section id), `empty_slots`
  (slots layout), `unknown_question_ids` (keys in your answers map that aren't questions or form
  fields: usually a typo).
