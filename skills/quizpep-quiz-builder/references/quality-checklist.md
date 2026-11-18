# Quizpep quiz quality checklist

Run through this before `create_quiz` / `update_quiz`, and again before `publish_quiz`.

## Ground it in the real catalog
- [ ] Called `get_store` first and used only its real **tags, product types, vendors and collections**.
      Never invent a tag like `oily-skin` because it sounds right; a value no product has silently
      matches nothing.
- [ ] Unsure what a tag means or which products carry it → `search_products` before mapping it.
- [ ] Matching ignores case and treats spaces/`-`/`_` alike, but is otherwise exact: `oily` ≠ `oily-skin`.
- [ ] Collections in `collection` outcomes/sections use the **handle**; `scope.collectionIds` and
      `fallback.collectionId` use the **id**. Product ids come from `search_products`.
- [ ] `scope` limits the quiz to the right products (e.g. only the skincare collection in a mixed store).
- [ ] `settings.fallback` set to a sensible collection or bestsellers, so a no-match still looks curated.

## Length and structure
- [ ] 3–7 questions for a product finder (5 is typical); up to ~12 for a personality quiz. Every question
      must change the result or collect something the merchant asked for; cut the rest.
- [ ] Easy, engaging question first; email/contact last (just before results), never first.
- [ ] 2–6 answers per choice question (≥2 is required). More than 6 → `multiple_choice`,
      `dropdown` or split the question.
- [ ] Multi-select questions say how many to pick ("Pick up to 2.") and set `maxSelections`; add an
      `exclusive` "None of these" when "none" is a real answer.
- [ ] Budget as a `slider` with `rangeRule` on `price` (`lte` + `filter`), or as choice answers with
      `require`/`prefer` signals if price bands map to tags.
- [ ] `resultCount` 3–6 (3 for focused finders, 6 for fashion/gifts); slots layout for routines.

## Mapping answers to products
- [ ] Every product-relevant answer has at least one signal. "No preference" answers may have none,
      on purpose.
- [ ] `prefer` for tastes (weight 4–5 for the decisive question, 2–3 for secondary ones); `require`
      only for true must-haves (size range, pet species, "for him/her"); `avoid` for exclusions
      (fragrance-free, allergens). Remember `prefer` never removes products.
- [ ] Each realistic answer combination still leaves products: no `require` + `require` combos on
      different questions that no product satisfies.
- [ ] Personality quizzes: every answer gives points to at least one personality; points are balanced
      so every personality can win; each personality has product signals (or pins) so its results differ.
- [ ] Manual results: ordered most specific first; a `default.outcome` covers everyone else.
- [ ] Branching: every branch ends at results; no jump loops back to an earlier question.

## Test before publishing (`test_quiz`)
- [ ] One answers map per main path (each answer of the most decisive question at least once), plus an
      "everything no preference" path and the strictest combination (all `require`s + lowest budget).
- [ ] `fallbackLevel` is 0 on every main path. 1+ means a dead end: loosen a `require`, fix a tag, or
      add products.
- [ ] Most returned items have `score` > 0 and sensible `reasons`; mostly-zero scores mean the signals
      don't match the catalog.
- [ ] Different answers give visibly different products (if every path returns the same 3 products, the
      mapping is too weak).
- [ ] Manual quizzes: each manual result is reached by its intended path (`resultId`), and a path that
      should fall through gets the default. Personality quizzes: each personality wins at least once
      (`personalityIds`).
- [ ] Out-of-stock products: `hideOutOfStock` is on unless the merchant says otherwise.

## Copy and tone
- [ ] Questions are short, conversational and in second person ("How does your skin feel by midday?").
- [ ] Answers are parallel, mutually exclusive and in shoppers' words, not catalog jargon or tag
      names ("Shiny all over", not "oily-skin").
- [ ] Match the store's voice; no hype, no medical or guaranteed-results claims.
- [ ] Intro says what the shopper gets and how long it takes ("3 questions, about a minute").
- [ ] Results heading/body explain why these products ("Matched to your skin and goals"); `showReasons` on.
- [ ] Every text exists in `defaultLocale`; add other locales only when the merchant sells in them.
- [ ] Email pages say what the email is for; marketing consent uses an explicit checkbox
      (`consent.enabled`), never pre-assumed. Link the privacy policy in a `footnote` when collecting emails.

## Images
- [ ] Every image URL is absolute `https://` (http shows as mixed content; relative paths fail validation).
- [ ] Prefer the merchant's own images; copy outside images into their Shopify Files with
      `upload_image` and use the returned CDN URL. Only use images the merchant may use.
- [ ] Never add product photos to the document: result cards take images, titles and prices from the
      catalog automatically.
- [ ] `image_choice` images: same aspect ratio and style across answers; set `imageAlt`/`alt` on intro,
      page media and section images. Keep `imageLabels: "below"` unless the picture alone is clear.

## Mobile and accessibility
- [ ] Answer labels short enough for a phone (≈ 40 characters max); long explanations go in `description`.
- [ ] Side images (`media.layout` left/right) are fine on mobile (they move on top); set
      `mobileImageUrl` or `mobile: "hidden"` for wide banners.
- [ ] `mobileColumns` 1–2 on product cards; text over images has enough contrast (use `overlay`).
- [ ] Don't convey meaning by colour or image alone; colours are `#rrggbb` with readable contrast.
- [ ] `customCss` only when asked; every selector prefixed with `.pq-root`.

## Before publishing
- [ ] `validate_quiz` returns zero errors; read each warning and fix it or tell the merchant why it stays.
- [ ] Discount codes in `settings.discount` already exist in Shopify Discounts (the quiz doesn't create them).
- [ ] `get_store` says the plan can publish; otherwise explain the paywall instead of calling `publish_quiz`.
- [ ] Ask the merchant to preview it on their storefront once (desktop and phone) after publishing.
