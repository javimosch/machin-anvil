# Rulepacks — deterministic expert lanes for anvil-serve

A **rulepack** (`.rules.json`) is a model-free expert lane: deterministic rules
that produce the same typed route/confidence/action response as a head lane.
No weights, no dependencies, ~0ms decisions (rules-only packs skip the trunk
forward entirely). A pack can also declare a fallback **head** evaluated only
when no rule fires — deterministic floor first, fuzzy phrasings second.

## Loading

    ANVIL_EXPERTS="mylane:/path/to/lane.rules.json,other:/path/x.head"

List paths inside a pack resolve relative to the pack file's directory.

## Format

```json
{
  "format": "rulepack/1",
  "classes": ["clean","suspicious","scam"],
  "default_class": "clean",
  "head": "/abs/or/relative/path/to/fallback.head",
  "rules": [
    {"class":"scam","kind":"regex","arg":"(won|lottery)","score":0.95},
    {"class":"suspicious","kind":"words","arg":"lists/words.txt","score":0.7}
  ]
}
```

- `classes` — the taxonomy; `default_class` is the verdict when nothing fires.
- `rules` — evaluated in order; **first hit wins**, its declared `score` becomes
  the confidence. `score` outside (0, 0.999] clamps to 0.999.
- `head` — optional `.head` fallback: only when the rules floor misses does the
  request pay a trunk forward pass.

## Rule kinds

| kind | arg | semantics |
|---|---|---|
| `regex` | ERE pattern | `regex_match` on the normalized text |
| `contains` | substring | literal `index() >= 0` |
| `words` | list file | word tokens `[a-z0-9']` → sorted-list membership |
| `digits` | `N` | a run of ≥9 digit chars exists |
| `luhn` | — | Luhn check on merged digit groups (`4111 1111 1111 1111` works) |
| `iban` | — | IBAN mod-97 on extracted candidates |
| `domains` | list file | domain tokens → sorted-list + parent-suffix walk |
| `ge` | `input:thresh` | `scores[input] >= thresh` (score-policy lanes) |
| `le` | `input:thresh` | `scores[input] <= thresh` |
| `missing` | `input` | fires when `scores` lacks the key (unchecked honesty) |

## Input deobfuscation

Before rule eval the text passes `deobfuscate()`: lowercase, unicode-lookalike
fold, `[at]`/` at `/` dot `-forms→`@`/`.`, leetspeak (`0→o 1→i 3→e 4→a 5→s
7→t`) inside alpha tokens, spaced-letter join, letter-flood squeeze. Digit-only
tokens are preserved, so `luhn`/`iban`/`digits` stay honest. `domains` and
`words` check BOTH deobfuscated and raw text (clean listed tokens stay
findable even when the fold would rewrite them).

## Score-policy lanes

`POST /v1/route {"state":"x","expert":"mylane","scores":{"nudity":0.8}}` —
`ge`/`le`/`missing` rules decide over upstream scorer output. This is the
honest image-moderation path: your scorer does pixels, mtlm owns the verdict.
Missing inputs surface in two ways: `missing` rules can route to a review
class, and **every** declared `ge`/`le` input absent from the request is named
in the response's `"unchecked":[...]` field.

## Response evidence

- `"matched_rule":"class:kind"` — which rule fired (absent on head-fallback
  and default verdicts).
- `"unchecked":["violence","self_harm"]` — declared score inputs not supplied.

## Conformal & actions

Rules verdicts ride the same pipeline: `ANVIL_EXPERT_MINCONF`,
`ANVIL_EXPERT_QHAT`, `delegate` on low conf, `tool_call` on pass. A pack's
`default_class` answer is what it returns when nothing fires (use `none`,
`clean`, `publish`, `oos` per your taxonomy).

## Authoring tips

- First rule wins — order from most-specific to most-general.
- Keep patterns anchored: `(verify.{0,10}account)` beats `verify` alone.
- List files: one entry per line, `#` comments, loaded + sorted at boot —
  hundred-thousand-entry blocklists run at ~0ms.
- Test evasion explicitly: the pack should catch `s.p.a.m`, `sp4m`, `[at]`
  forms — probe those, not just clean text.
