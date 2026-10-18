# Writing style: every text a person reads

Mosaicast core, the edge and the official plugins (bingo, stats, wiki, sample) all sound like one product.
A plugin's tile sits next to core's own text on the same page, so a plugin that writes differently stands
out. Follow this for `locales/*.json`, the `label`/`description` maps in `plugin.json`, consent purposes,
notification texts (`notifyText`) and anything else that reaches a screen. The source is
`mosaicast-core/docs/WRITING.md`; this file is its plugin copy.

Code comments, Javadoc, commit messages and docs may keep explaining the *why*. The filler words and dash
habits below are banned there too.

## The voice

**Friendly plain.** Write like someone who runs the site and explains it to a friend: short, warm, direct.
Plausible, Overcast and Basecamp are good models.

1. **Say what it does or what happens. Then stop.** Give a reason only when it changes what the reader does
   next. Background belongs in the docs, a code comment or the PR.
2. **Short sentences, one idea each.** Key word first. Active voice.
3. **Talk to the reader.** "you" in English, **"du" in German, everywhere** (visitors, players, podcasters,
   admins). Never "Sie". English uses contractions: can't, you'll, it's, don't, couldn't.
4. **Say the positive thing.** No "not X but Y", "X, not Y", "rather than", "instead of" set-ups.
5. **No dashes as punctuation.** No em dash (—), no en dash (–) between clauses, no " - ". Use a period or
   a comma. En dashes in ranges ("1–5") are fine.
6. **No examples in hints**, unless the format is not obvious (a URL, a time like 12:04). Then one, after a
   colon or "like".
7. **No history.** No "new", "now", "used to", "since 0.8". The changelog says what changed.
8. **No filler.** Never: simply, just, actually, exactly, really, deliberately, by design, note that,
   please, please note, the point is, ensure, seamless, robust, powerful, leverage, effortless.
   German: einfach, eigentlich, genau, bewusst, wirklich, bitte, selbstverständlich.
9. **Numbers over adjectives.** "Up to 64 characters" beats "a short name".
10. **Sentence case** for buttons, labels, headings and menu entries.

## Length budgets

| Kind | Budget |
|---|---|
| Button, menu entry | 1–3 words, verb first: "Save", "Hand in my bingo" |
| Label, column, heading | up to 4 words |
| Hint under a field, config `description` | one sentence, about 100 characters; two at most |
| Intro under a page title | one or two sentences |
| Confirm dialog | the action as a question, what goes with it, "You can't undo this." |
| Error | what happened, then what to do: "Couldn't save. Try again." |
| Empty state | what will show here, and how to get it there |

## Who reads it

- **Visitors** (tiles, pages, consent purposes): never say *plugin*, *slot*, *manifest* or *host*. Say
  *service*, *provider*, *stored on your device*. Name the company, not the hostname. Consent purposes are
  plain outcomes: "Remembers which stats you picked".
- **Podcasters and admins** (settings, admin panels, config `description`): *plugin*, *feed*, *episode*
  are fine. Name an environment variable only when the admin has to set it.
- **Consent and legal text** keeps the same voice and stays complete: every purpose, every company, how
  long, how to withdraw.

## German

- **du**, lower case. Imperatives short: "Versuch es noch mal", "Lies sie vor dem Speichern".
- Verbs over nouns. "Wird gespeichert, sobald du Play drückst" beats "Die Speicherung erfolgt bei
  Wiedergabe".
- **Folge**, not "Episode"; **Staffel**; Einstellungen; Hochladen. Keep the words German speakers use
  anyway: Podcast, Feed, Plugin, Tag, Login, Shownotes, Upload.
- Neutral words over gender marks: alle, wer, Hörende, Personen, Lesende. Use ":innen" only when nothing
  neutral exists.
- Errors: "X konnte nicht geladen werden." is fine; add what to do: "Versuch es noch mal."
- Translate the meaning, not the sentence.

## Before and after

| Before | After |
|---|---|
| Delete "{{title}}" and its {{count}} episodes? Their show notes, tags, everyone's listening position in them and anything plugins stored about them go too. The tag vocabulary stays, since other feeds may use it. This cannot be undone. | Delete "{{title}}" with all {{count}} episodes? Show notes, tags, listening progress and plugin data go with it. You can't undo this. |
| Your card is frozen — predictions are closed, so it counts as it stands. | Predictions are closed, so your card counts as it is. |
| No page matches. A link to a page that does not exist yet is fine — write it by hand and it shows as a red link. | No page matches. You can still link a page that doesn't exist yet. Type the link by hand and it shows up red. |
| How often the backend recomputes the highlight statistics, the page listing and the favourite counts. A saved change takes effect within one old interval — the backend does not have to be restarted. | How often the highlight stats, page list and favourite counts are recomputed. A change applies within one interval, without a restart. |
| Remembers which stats you picked (for example whether spoilers are shown) while this tab is open. Only written when you change the choice; holds no identifier. | Remembers which stats you picked, like whether spoilers show, while this tab is open. Only saved when you change it, and holds no identifier. |
| Diese Karte verrät, was in einer Folge passiert — bitte erst nach dem Hören öffnen. | Diese Karte verrät, was in der Folge passiert. Hör sie zuerst zu Ende. |

## Checklist before you commit a string

- Would a friend say it like this out loud?
- Can a word go without losing meaning? Then it goes.
- Any dash, "not … but", "rather than", "please", or a word from the filler list? Rewrite.
- Both locales changed, same keys, same placeholders, every plural form.
- Visitor text free of *plugin*, *slot*, *manifest*, *host*?
- Tests that match on text updated to the new wording (match a stable fragment, not the whole sentence).

A quick scan that catches most slips:

```sh
grep -nE '—| – | - |rather than|instead of|Please|please|cannot|could not|\bSie\b|\bIhr(e|en|em)?\b|z\. ?B\.|e\.g\.' frontend/locales/*.json
```
