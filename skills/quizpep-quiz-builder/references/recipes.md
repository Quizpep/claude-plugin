# Quizpep recipes

Six complete quiz documents. Each passes `validate_quiz` with zero errors. The only warning
they share is `/settings/fallback` ("No fallback products set"): set
`settings.fallback.collectionId` (or `productIds`) to a real collection/product id from `get_store` /
`search_products` to clear it.

Before you reuse one:
- Replace every tag, product type, vendor and collection handle with values that exist in the store
  (`get_store` lists them). The values below are placeholders that only illustrate the shape.
- Keep ids stable once a quiz is live (analytics, manual paths and customer tags refer to them).
- Run `test_quiz` with at least one answers map per main path (see quiz-format.md → Testing).

---

## (a) Product finder with tag signals

Use for the classic "find your product" quiz: every answer adds score to products with certain
tags, one answer removes products, a budget slider filters by price. The default when nothing
else is asked for.

```json
{
  "schemaVersion": 1,
  "title": {"en": "Skincare finder"},
  "defaultLocale": "en",
  "locales": ["en"],
  "settings": {
    "scope": {"type": "all"},
    "resultCount": 3,
    "hideOutOfStock": true,
    "tieBreak": "best_selling"
  },
  "intro": {
    "heading": {"en": "Find your skincare match"},
    "body": {"en": "Three quick questions, <strong>about a minute</strong>."},
    "buttonLabel": {"en": "Start"}
  },
  "questions": [
    {
      "id": "skin_type",
      "type": "single_choice",
      "title": {"en": "How does your skin feel by midday?"},
      "propertyName": "skin_type",
      "answers": [
        {"id": "oily", "label": {"en": "Shiny all over"}, "customerTag": "quiz-skin-oily",
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["oily-skin"], "weight": 4}]},
        {"id": "dry", "label": {"en": "Tight or flaky"}, "customerTag": "quiz-skin-dry",
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["dry-skin"], "weight": 4}]},
        {"id": "normal", "label": {"en": "Comfortable"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["all-skin-types"], "weight": 2}]}
      ]
    },
    {
      "id": "concerns",
      "type": "multiple_choice",
      "title": {"en": "What would you like to improve?"},
      "description": {"en": "Pick up to 2."},
      "minSelections": 1,
      "maxSelections": 2,
      "answers": [
        {"id": "acne", "label": {"en": "Breakouts"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["acne", "blemish"]}]},
        {"id": "aging", "label": {"en": "Fine lines"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["anti-aging"]}]},
        {"id": "fragrance_free", "label": {"en": "I avoid fragrance"},
         "signals": [{"kind": "avoid", "attribute": {"source": "tag"}, "values": ["fragranced"]}]},
        {"id": "none", "label": {"en": "None of these"}, "exclusive": true}
      ]
    },
    {
      "id": "budget",
      "type": "slider",
      "title": {"en": "Your budget per product?"},
      "range": {"min": 10, "max": 100, "step": 5, "unit": {"en": "$"}},
      "rangeRule": {"attribute": {"source": "price"}, "operator": "lte", "mode": "filter"}
    }
  ],
  "results": {
    "layout": "grid",
    "heading": {"en": "Your top picks"},
    "body": {"en": "Matched to your skin and goals."},
    "showReasons": true,
    "sections": [
      {"id": "header", "type": "header"},
      {"id": "products", "type": "products", "card": {"columns": 3, "mobileColumns": 2}},
      {"id": "footer", "type": "footer"}
    ]
  }
}
```

---

## (b) Email capture with consent, discount and results email

Use when the merchant wants leads: a form page collects first name + email with a marketing
checkbox, the discount is only shown to shoppers who gave their email, and the results are
emailed. The email is sent to everyone who typed an email; only shoppers who tick the box are
subscribed to marketing.

```json
{
  "schemaVersion": 1,
  "title": {"en": "Coffee match"},
  "defaultLocale": "en",
  "locales": ["en"],
  "settings": {
    "scope": {"type": "all"},
    "resultCount": 3,
    "discount": {
      "code": "QUIZ10",
      "message": {"en": "10% off your first bag, applied at checkout."},
      "placement": "results",
      "requireEmail": true,
      "autoApply": true
    },
    "resultsEmail": {
      "enabled": true,
      "subject": {"en": "{{first_name|Your}} coffee matches"},
      "heading": {"en": "Here are your coffees, {{first_name|friend}}"},
      "body": {"en": "Picked from your answers. Your code is inside."},
      "showDiscount": true,
      "showAnswers": true
    }
  },
  "questions": [
    {
      "id": "brew",
      "type": "single_choice",
      "title": {"en": "How do you brew?"},
      "answers": [
        {"id": "espresso", "label": {"en": "Espresso"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["espresso"]}]},
        {"id": "filter", "label": {"en": "Filter or pour-over"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["filter"]}]}
      ]
    },
    {
      "id": "roast",
      "type": "single_choice",
      "title": {"en": "Which roast do you like?"},
      "answers": [
        {"id": "light", "label": {"en": "Light and fruity"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["light-roast"], "weight": 4}]},
        {"id": "dark", "label": {"en": "Dark and chocolatey"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["dark-roast"], "weight": 4}]}
      ]
    },
    {
      "id": "contact",
      "type": "form",
      "title": {"en": "Where should we send your matches?"},
      "description": {"en": "We'll email your results and a 10% code."},
      "fields": [
        {"id": "first_name", "type": "text", "label": {"en": "First name"}, "propertyName": "first_name"},
        {"id": "email", "type": "email", "label": {"en": "Email"}, "required": true}
      ],
      "consent": {"enabled": true, "label": {"en": "Email me coffee news and offers"}},
      "footnote": {"en": "Unsubscribe any time. See our <a href=\"/policies/privacy-policy\">privacy policy</a>."}
    }
  ],
  "results": {
    "layout": "grid",
    "heading": {"en": "Your coffees, {{first_name|friend}}"},
    "body": {"en": "Roasted to match how you brew."}
  }
}
```

---

## (c) Personality quiz

Use for "which type are you?" quizzes: answers give points to personalities, the one with the
most points wins, its title and description replace the results heading and text, and its own
signals pick the products. Ties go to the personality listed first (no `personalityRules`).

```json
{
  "schemaVersion": 1,
  "title": {"en": "What's your scent personality?"},
  "defaultLocale": "en",
  "locales": ["en"],
  "settings": {"scope": {"type": "all"}, "resultCount": 3},
  "questions": [
    {
      "id": "weekend",
      "type": "single_choice",
      "title": {"en": "Your ideal weekend?"},
      "answers": [
        {"id": "beach", "label": {"en": "A day at the beach"},
         "signals": [{"kind": "personality", "personalityId": "fresh", "points": 2}]},
        {"id": "garden", "label": {"en": "Pottering in the garden"},
         "signals": [{"kind": "personality", "personalityId": "floral", "points": 2}]},
        {"id": "cabin", "label": {"en": "A cabin by the fire"},
         "signals": [{"kind": "personality", "personalityId": "warm", "points": 2}]}
      ]
    },
    {
      "id": "colour",
      "type": "single_choice",
      "title": {"en": "Pick a colour"},
      "answers": [
        {"id": "blue", "label": {"en": "Sea blue"},
         "signals": [{"kind": "personality", "personalityId": "fresh"}]},
        {"id": "pink", "label": {"en": "Blush pink"},
         "signals": [{"kind": "personality", "personalityId": "floral"}]},
        {"id": "amber", "label": {"en": "Amber"},
         "signals": [{"kind": "personality", "personalityId": "warm"}]}
      ]
    },
    {
      "id": "moments",
      "type": "multiple_choice",
      "title": {"en": "When will you wear it?"},
      "maxSelections": 2,
      "answers": [
        {"id": "office", "label": {"en": "At work"},
         "signals": [{"kind": "personality", "personalityId": "fresh"}]},
        {"id": "date", "label": {"en": "Date night"},
         "signals": [{"kind": "personality", "personalityId": "warm"}, {"kind": "personality", "personalityId": "floral"}]},
        {"id": "everyday", "label": {"en": "Every day"},
         "signals": [{"kind": "personality", "personalityId": "floral"}]}
      ]
    }
  ],
  "results": {
    "layout": "personality",
    "heading": {"en": "Your scent personality"},
    "personalities": [
      {
        "id": "fresh",
        "title": {"en": "The Free Spirit"},
        "description": {"en": "<p>Clean, airy and bright. You love <strong>citrus and sea salt</strong>.</p>"},
        "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["fresh", "citrus"], "weight": 5}]
      },
      {
        "id": "floral",
        "title": {"en": "The Romantic"},
        "description": {"en": "<p>Soft and blooming: rose, peony and jasmine.</p>"},
        "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["floral"], "weight": 5}]
      },
      {
        "id": "warm",
        "title": {"en": "The Cosy Soul"},
        "description": {"en": "<p>Amber, vanilla and woods that linger.</p>"},
        "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["warm", "woody"], "weight": 5}]
      }
    ],
    "sections": [
      {"id": "header", "type": "header"},
      {"id": "products", "type": "products"},
      {"id": "footer", "type": "footer"}
    ]
  }
}
```

To show fixed products per personality instead, give the personality `pin` signals
(`{"kind": "pin", "productId": "gid://shopify/Product/<id>"}`) with real ids from `search_products`;
pinned products are shown in that order and replace the scored ranking.

Give each personality an `imageUrl` (absolute https; `upload_image` first) to show its picture
above the heading on the results page.

**"Which one is right for you?"** (good/better/best): one personality per product tier, set
`settings.resultCount` to 1 so only the best match shows, and put the products section in a
narrow column (`"style": {"width": "narrow"}`). A `highlights` section with `level` 1–5 per tier
makes a quick "how they compare" row.

---

## (d) Manual results with path conditions

Use when the merchant wants to decide exactly what each answer combination gets ("beginners
who ride powder get the starter kit collection"). Manual results are checked top to bottom;
the first whose path matches wins; otherwise `results.default.outcome` applies.

```json
{
  "schemaVersion": 1,
  "title": {"en": "Find your board"},
  "defaultLocale": "en",
  "locales": ["en"],
  "settings": {"scope": {"type": "all"}, "resultCount": 4},
  "questions": [
    {
      "id": "level",
      "type": "single_choice",
      "title": {"en": "How often do you ride?"},
      "answers": [
        {"id": "beginner", "label": {"en": "I'm just starting"}},
        {"id": "regular", "label": {"en": "A few times a season"}},
        {"id": "expert", "label": {"en": "Every weekend"}}
      ]
    },
    {
      "id": "terrain",
      "type": "multiple_choice",
      "title": {"en": "Where do you ride?"},
      "answers": [
        {"id": "groomers", "label": {"en": "Groomed runs"}},
        {"id": "powder", "label": {"en": "Powder"}},
        {"id": "park", "label": {"en": "The park"}}
      ]
    },
    {
      "id": "budget",
      "type": "number",
      "title": {"en": "Your budget in dollars?"},
      "range": {"min": 0, "max": 2000, "unit": {"en": "$"}},
      "rangeRule": {"attribute": {"source": "price"}, "operator": "lte"}
    }
  ],
  "results": {
    "layout": "grid",
    "mode": "manual",
    "heading": {"en": "Boards for you"},
    "manual": [
      {
        "id": "starter",
        "name": "Beginners",
        "path": [{"questionId": "level", "answerIds": ["beginner"]}],
        "outcome": {"type": "collection", "collection": "starter-kits"},
        "heading": {"en": "Start here"},
        "body": {"en": "Forgiving boards that make learning easy."}
      },
      {
        "id": "powder_pro",
        "name": "Experts who ride powder",
        "path": [
          {"questionId": "level", "answerIds": ["regular", "expert"]},
          {"questionId": "terrain", "answerIds": ["powder"]}
        ],
        "outcome": {"type": "collection", "collection": "powder-boards"},
        "heading": {"en": "Built for deep days"}
      },
      {
        "id": "small_budget",
        "name": "Under $300",
        "path": [{"questionId": "budget", "max": 300}],
        "outcome": {"type": "text"},
        "heading": {"en": "Have a look at our rentals"},
        "body": {"en": "Our boards start at $300. <a href=\"/pages/rentals\">Rent one</a> for the season instead."}
      }
    ],
    "default": {"outcome": {"type": "automatic"}}
  }
}
```

Notes: a path condition with several `answerIds` is OR (`"match": "all"` makes it AND for
multiple choice). Number questions use `min`/`max`. A `products` outcome needs real ids:
`{"type": "products", "productIds": ["gid://shopify/Product/<id>"]}`.

---

## (e) Branching / skip logic

Use when some questions only make sense for some shoppers. Answer `next` wins over question
`next`; without either the next question in order follows; the last question leads to results.

Flow: `who` → (me) `style` → `size` → results; (gift) `recipient` → `gift_budget` → results;
(browsing) results straight away.

```json
{
  "schemaVersion": 1,
  "title": {"en": "Find the perfect piece"},
  "defaultLocale": "en",
  "locales": ["en"],
  "settings": {"scope": {"type": "all"}, "resultCount": 6},
  "questions": [
    {
      "id": "who",
      "type": "single_choice",
      "title": {"en": "Who are you shopping for?"},
      "answers": [
        {"id": "me", "label": {"en": "Myself"}},
        {"id": "gift", "label": {"en": "It's a gift"},
         "next": {"to": "question", "questionId": "recipient"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["gift"], "weight": 2}]},
        {"id": "browsing", "label": {"en": "Just browsing"},
         "next": {"to": "results"},
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["bestseller"]}]}
      ]
    },
    {
      "id": "style",
      "type": "image_choice",
      "title": {"en": "Which look is most you?"},
      "imageLabels": "below",
      "answers": [
        {"id": "classic", "label": {"en": "Classic"}, "imageUrl": "https://images.unsplash.com/photo-1490481651871-ab68de25d43d",
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["classic"]}]},
        {"id": "street", "label": {"en": "Streetwear"}, "imageUrl": "https://images.unsplash.com/photo-1523398002811-999ca8dec234",
         "signals": [{"kind": "prefer", "attribute": {"source": "tag"}, "values": ["streetwear"]}]}
      ]
    },
    {
      "id": "size",
      "type": "dropdown",
      "title": {"en": "Your usual size?"},
      "next": {"to": "results"},
      "answers": [
        {"id": "s", "label": {"en": "S"}, "signals": [{"kind": "variant_hint", "option": "Size", "value": "S"}]},
        {"id": "m", "label": {"en": "M"}, "signals": [{"kind": "variant_hint", "option": "Size", "value": "M"}]},
        {"id": "l", "label": {"en": "L"}, "signals": [{"kind": "variant_hint", "option": "Size", "value": "L"}]}
      ]
    },
    {
      "id": "recipient",
      "type": "single_choice",
      "title": {"en": "Who is it for?"},
      "answers": [
        {"id": "her", "label": {"en": "Her"},
         "signals": [{"kind": "require", "attribute": {"source": "collection"}, "values": ["women"]}]},
        {"id": "him", "label": {"en": "Him"},
         "signals": [{"kind": "require", "attribute": {"source": "collection"}, "values": ["men"]}]}
      ]
    },
    {
      "id": "gift_budget",
      "type": "slider",
      "title": {"en": "What would you like to spend?"},
      "range": {"min": 20, "max": 200, "step": 10, "unit": {"en": "$"}},
      "rangeRule": {"attribute": {"source": "price"}, "operator": "closest", "mode": "score"}
    }
  ],
  "results": {
    "layout": "grid",
    "heading": {"en": "Picked for you"}
  }
}
```

Replace the Unsplash URLs with the merchant's own images (`upload_image` copies an https URL into
their Shopify Files). Answer-level `next` only works on single-select questions
(`single_choice`, `dropdown`, single `image_choice`); for `multiple_choice` use question `next`.

---

## (f) Couples quiz

Use only when the merchant asks for a quiz two partners take and compare (relationship,
wedding, compatibility). With `settings.couples.enabled` the widget replaces the normal flow:
shoppers choose a mode, each partner answers privately, and both see a comparison map grouped
by category. Only choice questions are compared.

```json
{
  "schemaVersion": 1,
  "title": {"en": "Are we aligned?"},
  "defaultLocale": "en",
  "locales": ["en"],
  "settings": {
    "scope": {"type": "all"},
    "resultCount": 3,
    "couples": {
      "enabled": true,
      "modes": ["solo", "invite", "together"],
      "predictions": true,
      "inviteDays": 14,
      "categories": [
        {"id": "future", "title": {"en": "Our future"},
         "prompts": {"en": "<ul><li>Where do you see us in five years?</li><li>What would make home feel like home?</li></ul>"}},
        {"id": "everyday", "title": {"en": "Everyday life"}}
      ]
    }
  },
  "intro": {
    "heading": {"en": "Are we aligned?"},
    "body": {"en": "Answer privately, then compare. Your answers are only shared if you both agree."},
    "buttonLabel": {"en": "Begin"}
  },
  "questions": [
    {
      "id": "kids",
      "type": "single_choice",
      "title": {"en": "Do you want children?"},
      "category": "future",
      "weigh": true,
      "answers": [
        {"id": "yes", "label": {"en": "Yes"}},
        {"id": "no", "label": {"en": "No"}},
        {"id": "unsure", "label": {"en": "Not sure yet"}, "unknown": true}
      ]
    },
    {
      "id": "home",
      "type": "single_choice",
      "title": {"en": "Where would you like to live?"},
      "category": "future",
      "answers": [
        {"id": "city", "label": {"en": "In the city"}},
        {"id": "country", "label": {"en": "In the countryside"}},
        {"id": "abroad", "label": {"en": "Abroad"}}
      ]
    },
    {
      "id": "weekends",
      "type": "multiple_choice",
      "title": {"en": "How do you like to spend weekends?"},
      "category": "everyday",
      "maxSelections": 2,
      "answers": [
        {"id": "out", "label": {"en": "Out with friends"}},
        {"id": "in", "label": {"en": "A quiet night in"}},
        {"id": "active", "label": {"en": "Something active"}}
      ]
    }
  ],
  "results": {
    "layout": "grid",
    "heading": {"en": "Your compatibility map"}
  }
}
```
