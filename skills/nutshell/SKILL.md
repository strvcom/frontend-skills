---
name: nutshell
description: Distill anything to its absolute essence — the fewest words that carry the full meaning. Use when user wants a TLDR, asks to summarize, name, or condense something, or mentions "nutshell".
---

# Nutshell

A lens for finding the essence of anything. Useful when distillation helps — TLDR a concept, name a thing, condense a procedure, find an analogy.

## The operation

Study the subject. Find what carries the meaning — what would break if removed. Output only that, in the fewest words, in the register of the domain (technical, business, casual — match the subject).

**The aha test:** the output must instantly form the correct mental model in the reader's mind. They should go "oh, *that's* what it is." If they don't, it's not tight enough yet.

**Every word earns its place:** cut anything that doesn't carry meaning. But never sacrifice meaning for brevity — a short wrong answer is worse than a longer right one.

**Deliver, don't ask:** no clarifying questions, one committed answer (not options), analogies only when they hold up under scrutiny.

## Output style

Extremely concise. Sacrifice grammar for concision. Practical, elegant. Short, memorable phrasing.

Lead with the subject — the output should stand alone, understandable without seeing the question. Example: "STRV is a Czech-born dev agency that builds apps for Silicon Valley." not "Czech-born dev agency that builds apps for Silicon Valley."

Output should invite reading — like a well-organized student notebook. Visual hierarchy mirrors meaning, white space lets ideas breathe. Form is part of the message: a wall of text repels, a tidy page draws the reader in.

- **Bold** the subject and key anchor words
- Let white space do work — sections breathe, points separate

## Structure

Three parts, in this order:

1. **The core** (always) — one blockquote (`>`) line that passes the aha test alone. If the reader stops here, they still have the right mental model.
2. **Bullets** (if necessary) — the few key points that hold the core up. Parallel, short, one idea each. Skip them when the core already says it all.
3. **A drawing** (if appropriate) — one small ASCII drawing that shows the shape of the subject: flow, timeline, structure, state, contrast.

### The drawing

Don't be shy about it. It doesn't have to be strictly technical. A drawing that only makes the nutshell easier to see and remember is reason enough.

- **Glance test:** understood in ~2 seconds
- **Small:** ≤ ~50 columns, ≤ ~8 lines, in a code block so it renders monospaced
- **Craft:** box-drawing characters (`┌─┐│└┘├┤ ─▶ ◀─ ▼ ▲ ●`), arrows labeled with what travels along them
- **No emoji inside boxes or aligned rows** — double width breaks alignment
- **Never forced:** if the subject has no shape and a drawing adds nothing, leave it out
- **Complements the bullets, doesn't repeat them:** bullets say *what*, the drawing shows *shape* (when, where, how it flows)

## Examples

### `/nutshell photosynthesis`

> **Photosynthesis**: a plant makes **sugar** from **light**, **water** and **CO₂**, and vents **oxygen** as waste.

- **Where:** in leaf chloroplasts; green **chlorophyll** catches the light
- **Two phases:** light splits water (energy + O₂), then that energy builds sugar from CO₂
- **Why it matters:** nearly every food chain and all the oxygen in the air depend on it

```
           light
             │
             ▼
  CO₂ ──▶ ┌──────┐ ──▶ sugar
  H₂O ──▶ │ leaf │ ──▶ O₂ (waste)
          └──────┘
```

### `/nutshell https://en.wikipedia.org/wiki/Adolf_Hitler`

> **Adolf Hitler** (1889–1945): Nazi dictator of Germany who started **WWII** and orchestrated the **Holocaust**.

- **Rise:** failed painter → WWI soldier → NSDAP leader → chancellor → Führer
- **Ideology:** racism, antisemitism, "living space" in the East
- **Czechoslovakia:** Munich Agreement and occupation → Protectorate of Bohemia and Moravia
- **Holocaust:** ~6M Jews and millions of others murdered

```
 1889      1933       1938-39            1945
  ●─────────●────────────●─────────────────■
 Austria  chancellor  Munich, Poland   suicide
                      → war            in Berlin
```
