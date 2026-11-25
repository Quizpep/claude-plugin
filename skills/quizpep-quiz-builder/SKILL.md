---
name: quizpep-quiz-builder
description: Build, edit, test and publish Shopify product quizzes in Quizpep through the Quizpep connector. Use when someone asks to create a product quiz, product finder, recommendation quiz, personality quiz, gift finder, email sign-up quiz or couples quiz for their Shopify store, or to change, fix, translate, restyle or publish an existing Quizpep quiz.
---

# Quizpep quiz builder

Quizpep is a Shopify app that shows product quizzes on a store. You build quizzes for the merchant
with the **Quizpep connector** tools (`get_store`, `create_quiz`, ...). A quiz is one JSON document;
the connector checks it with the same validator as the Quizpep editor.

If the tools aren't available, the connector isn't connected. Tell the merchant to connect it: in
Claude, Settings → Connectors → Add custom connector → `https://app.quizpep.com/mcp`; in Claude Code,
`/plugin marketplace add quizpep/claude-plugin`, `/plugin install quizpep@quizpep`, then `/mcp` to sign in.
They need Quizpep installed on their store.

## Ground rules

- **Drafts only, unless asked.** `create_quiz` and `update_quiz` save drafts; shoppers see nothing.
  Call `publish_quiz` only when the merchant explicitly asks to publish *this* quiz in this
  conversation. Never publish "to test it".
- **The store's real catalog only.** Every tag, product type, vendor and collection in a signal must
  come from `get_store` (or `search_products`). Never invent a tag, and never tell the merchant to
  add tags unless they ask how to improve matching.
- **Change, don't replace.** For an existing quiz: `get_quiz`, change what was asked, send the whole
  document back with `update_quiz`. Keep ids stable (analytics and saved answers use them).
- **Plain language with the merchant.** They are store owners, not developers. Talk about questions,
  answers and results, not JSON, ids or signals, unless they're clearly a developer.
- **Don't publish personal data or claims you can't back up.** No medical, legal or financial promises
  in quiz copy; supplement and skincare quizzes keep a short "not medical advice" line.

## Workflow: a new quiz

1. **Learn the store.** `get_store`: catalog values, plan (`can_publish`), existing quizzes.
   `search_products` for the product lines the quiz is about (see real titles, tags, prices).
2. **Ask only what you can't infer** (at most 2–3 questions, in one message): the quiz's goal
   (recommend products, grow the email list, both), who takes it, and whether to ask for an email.
   If the merchant already said, don't ask.
3. **Start from a template.** `list_templates`; if one is close, `get_template` and adapt it. It is
   already valid and well designed, with photos, icons and a matching results page. There are product
   finders for many niches, personality quizzes (style, scent, self-care, décor, eyewear), a size
   finder, a bundle builder, a gift finder, an email-capture quiz and a "which one is right for you?"
   good/better/best quiz. Use `blank` only for something unusual.
4. **Write the quiz.** Usually 4–8 questions, 2–6 answers each. For each answer, decide which
   products it should lead to and express that as signals on real catalog values
   (`references/quiz-format.md` §9). Match the store's tone and colors (`design`).
5. **Check.** `validate_quiz`. Fix every error; read the warnings (fix the ones that matter, like a
   missing fallback). Repeat until `valid` is true.
6. **Save.** `create_quiz` with a clear name. Keep the `quiz_id`.
7. **Test the paths.** `test_quiz` with 3–5 realistic answer sets, including the extremes. For each:
   products should match the answers (`score` > 0, sensible `reasons`), `fallback_level` 0, and
   different answers should give different products. Fix the signals and `update_quiz` if not.
8. **Hand over.** Summarise the quiz in a few lines (questions, what each path recommends, email or
   discount), and give the `editor_url` so they can preview it and fine-tune the look. Offer to
   publish. If `can_publish` is false, say publishing needs the Scale plan (link from `get_store`).
9. **Publish when asked.** `publish_quiz`. Then give the `page_url`, and say that to show it on a
   product, collection or home page they add the Quizpep **Quiz** block in the theme editor.

## Workflow: change an existing quiz

`list_quizzes` → `get_quiz` → change → `validate_quiz` → `update_quiz` → `test_quiz` if matching
changed. A quiz with status `changed` has saved changes that aren't live; they go live only with
`publish_quiz` (when asked). Mention it if the quiz is live.

## Things that trip people up

- **No empty objects.** Never send `{}` for an optional object (`style`, `card`, `discount`, ...):
  it's rejected. Leave the key out.
- **URLs are absolute https.** `imageUrl`, `link`, `redirectUrl` must be full `https://` links (a
  relative `/collections/all` is rejected). Links inside rich text may be relative.
- **Images.** Use `upload_image` to put an image from a link into the store's Shopify Files, then use
  the returned URL. Only use images the merchant gave you or has the right to use. Product photos
  come from the catalog automatically. Template photos are copied to the store on save.
- **`customCss` isn't scoped for you.** Prefix every selector with `.pq-root`. Prefer `design`
  settings over CSS.
- **`prefer` never removes products**; `require` does. A quiz with only `prefer` signals always fills
  its slots, so check `score`/`reasons` in `test_quiz`, not just `fallback_level`.
- **Collections:** signals, `outcome.collection` and collection sections use the collection
  **handle**; `settings.scope.collectionIds` and `settings.fallback.collectionId` need the numeric id
  or gid.
- **Branching:** answer-level `next` works on single-choice questions only. Don't build loops.
- **Range rules:** only `price` is reliable today. Don't filter on metafields.
- **Personality quizzes** share one results page (`results.sections`) for every personality; the
  personality's own title and description fill the header.
- **Results email** goes to everyone who typed an email; the consent box only controls marketing.
- **Big quizzes:** `update_quiz` takes the whole document. For a large change, build it fully, then
  validate once.

## Errors from the tools

Tool errors come back as text with `error` and, for documents, `errors[]` / `warnings[]` with a JSON
`path` (e.g. `/questions/2/answers/1/id`). Fix and retry; don't ask the merchant about schema errors.
If the store isn't connected or Quizpep was uninstalled, the tools stop working: ask the merchant to
reconnect.

## References (read when needed)

- `references/quiz-format.md`: the whole document format, signals and matching, results, sections,
  design, couples, validation rules and the `test_quiz` answers format. Read §9 and §10 before
  writing signals and results.
- `references/recipes.md`: six complete, valid example quizzes (product finder, email + discount,
  personality, hand-picked results with conditions, branching, couples). Copy the closest one's
  structure.
- `references/quality-checklist.md`: what makes a quiz good; run through it before handing over.
- `get_quiz_format` (tool): the exact schema of any part, e.g. `{"part": "ResultSection"}`, for
  fields the references don't cover.
