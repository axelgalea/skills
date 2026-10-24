# AGENTS.md

## Package manager

Use the package manager the project already uses. The lockfile in the project root tells you which one. If there is no lockfile, use nub.

## Stack

- Effect v4 for application code.
- `@effect/vitest` for tests.
- Effect Atom for state in web apps. Atoms come from `effect/unstable/reactivity` and React hooks come from `@effect/atom-react`. Do not use the v3 packages `@effect-atom/atom` or `@effect-atom/atom-react`. Do not add another state library such as Redux, Zustand, Jotai, or TanStack Query.

Effect v4 changed many APIs from v3, and most examples online and in model training data are v3. Check the source in `repos/effect` before you use an API you are not sure about.

## Library source in `repos/`

Keep shallow clones of the main external libraries in `repos/` at the project root. Search them with `rg` instead of guessing how a library works from memory.

```bash
git clone --depth 1 https://github.com/Effect-TS/effect-smol repos/effect
git clone --depth 1 https://github.com/TanStack/router repos/tanstack-router

rg "export const catchTag" repos/effect/packages/effect/src
rg -l "createFileRoute" repos/tanstack-router/packages
```

- Add `repos/` to `.gitignore`.
- Do not edit files in `repos/` or import from them.
- If a library you need to read is missing, clone it the same way.

## Code

- Do not extract single-use helpers. Inline the logic at the call site unless the helper is reused, hides a complex boundary, or has a name that makes the caller easier to read.
- Validate unknown values once, at the boundary that owns them. Pass typed values inward. Do not repeat `typeof value === "object"` or property checks on values a schema, constructor, or type already guarantees.
- Avoid `try`/`catch` where possible.
- Do not use `any`.
- Let TypeScript infer types. Write annotations or interfaces only for exports or when inference makes the code unclear.
- Prefer `flatMap`, `filter`, and `map` over `for` loops. Use a type guard in `filter` so the narrowed type carries through.
- In Effect generators, bind a service to a named variable before you call its methods. Do not write `yield* (yield* Foo.Service).bar()`.

## Scope

Before you add complexity for an unlikely race condition or security edge case, tell the user the concrete failure, how likely it is, and what the fix costs in complexity. Wait for their answer. Do not expand scope on your own for theoretical robustness.

## Tests

- Avoid mocks. Do not touch `globalThis` unless there is no other option.
- Test the real implementation. Do not copy logic into a test to compute the expected value.
- In a monorepo, run tests from the package directory, not the repo root.

## Writing

These rules apply to everything you write: replies, docs, code comments, commit messages, and PR descriptions.

### Say it plainly

- Say what the thing does. Name the mechanism, the command, or the number. If a sentence could appear unchanged in another project's docs, cut it.
- Write one idea per sentence. Split any sentence the reader has to parse twice.
- Use active voice and name the actor. Write "the loader parses the file", not "the file is parsed".
- Use the plain word. Write "use", "help", "many", "if", "is", and "has" instead of "utilize", "leverage", "facilitate", "numerous", "in the event that", "serves as", and "boasts".
- Cut adverbs. Use a stronger verb or the measured number.
- Write whole sentences with articles and verbs. Spell out arrows and abbreviations.
- Say it literally. No aphorisms, personified code, figurative verbs, or metaphor nouns such as substrate, vector, primitive, surface, harness, scaffolding, paradigm, north star, or flywheel.

### Cut

- AI vocabulary: additionally, crucial, delve, enduring, enhance, foster, garner, interplay, intricate, landscape, pivotal, showcase, tapestry, testament, underscore, vibrant.
- Trailing "-ing" phrases such as "highlighting...", "ensuring...", and "showcasing...".
- "Not just X, but Y". State the point.
- "From X to Y" when X and Y are not on a real scale. List the items.
- Vague attributions such as "experts believe". Name the source or cut the claim.
- Filler. "In order to" becomes "to", "due to the fact that" becomes "because", and "it is important to note that" goes away.
- Stacked hedges. One "may" is enough.
- Generic conclusions. End on a specific fact or next step.
- Chatbot and sycophantic phrases such as "Great question", "Certainly!", "I hope this helps", and "Let me know if...".

### Structure

- Use the natural number of items. Do not force groups of three.
- Pick one name for a thing and repeat it. Do not cycle synonyms.

### Formatting

- No em dashes. Do not substitute en dashes, hyphens, or parentheses. End the sentence or use a comma.
- Use a colon only before a list or an example, never to join two clauses.
- Bold rarely. Do not bold every proper noun or acronym.
- Do not write lists where a bold label and colon restate the line. Write prose instead. A bold lead-in that ends in a period and is followed by new detail is fine.
- Use sentence case for headings.
- No decorative emojis.
- Use straight quotes.
