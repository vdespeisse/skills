---
name: ui-design
description: Build an interactive design demo in a Vue / Nuxt UI app — several presets of one screen or section, switchable from a dev-only floating panel with customisation controls and a "Copy settings" button whose JSON the user pastes back to pick. Use when the user asks for options, proposals, variants or a design demo for a piece of UI ("/ui-design give me options for X").
---

# UI design demo

Throwaway scaffolding that lets the user **see several designs of one piece of UI in place, on real data,
tweak them, and hand back their choice as JSON**. Everything it adds is deleted once they choose; only the
chosen design is kept, written properly.

The loop: clarify → build presets + controls → verify in the browser → hand over → the user pastes their
settings → fold the choice in and delete the rest.

Assumes a Vue 3 app using [Nuxt UI](https://ui.nuxt.com) components (`UCard`, `URadioGroup`, `USwitch`,
`UButton`, …), in Nuxt or in plain Vue + Vite. [PANEL.md](PANEL.md) has the templates and what to change for
a non-Nuxt app.

## 0. Learn the project

Before anything else, find:

- **The design rules**: a `DESIGN.md`, a style guide, the UI section of `CLAUDE.md` / `AGENTS.md`, or the
  conventions the existing screens follow (type scale, text colours, spacing, how tables and actions work).
  Every preset must follow them.
- **How to run the app**: the dev script, and any project-specific way to start the stack or seed data.
  Use what the project documents; don't start pieces by hand when a script does it.
- **How UI is tested**: the typecheck command (`npx nuxi typecheck`, `vue-tsc --noEmit`, …), the unit test
  runner, and any e2e suite.
- **Where code lives**: the components, composables and utils directories (`app/` in Nuxt 4, the root or
  `src/` elsewhere) and the component naming convention.

## 1. Clarify before building

Read the request, the page it targets and the design rules. Then settle these with the user
(`AskUserQuestion`), **skipping any the request already answers**:

1. **What is being evaluated**, which decides what the presets disagree about:
   - **UX / structure**: what is shown, in which order, how it's grouped and how much weight each part
     gets. Presets differ in layout and information hierarchy. Visual style stays the app's.
   - **UI / visual**: typography, colour, spacing, density, borders vs tints, radius. Presets share one
     structure and differ in look. Customisations are mostly visual knobs.
   - **Both**: structural presets, plus visual knobs on top.
2. **Scope**: the whole page, or one section of it (e.g. "the summary at the top of the compare page").
   Everything outside the scope stays untouched, so the presets are judged against the real surroundings.
3. **Presets and knobs**: propose the list yourself and let the user adjust it. Don't ask them to invent it.
   - 3–4 presets, labelled `A · <what it is>`.
   - A handful of customisations, each either a choice (2–4 values) or a toggle. Say which apply to all
     presets and which to one.

   Good knobs are the questions you'd otherwise ask in text, such as "show the interval?", "only the lines
   that differ?", "percent or fraction?", or "outline, tint or accent edge?".

If the user is unreachable, pick sensible answers, state them in one line and build.

## 2. Build it

Put the scaffolding where the real code lives. Name it so it's obviously throwaway, and gate the panel on
the dev flag (`import.meta.dev` in Nuxt, `import.meta.env.DEV` in Vite) so a production or e2e build never
shows it.

| Piece | Where |
| --- | --- |
| Pure helpers the presets share (counts, wording, formatting) | `utils/<topic>.ts` + a test: these usually survive the choice, so write them properly (test-first) |
| One component per preset | `components/<area>/<topic>/` (e.g. `comparison/summary/Banner.vue`) |
| A wrapper that renders the chosen preset | `components/<area>/<topic>/<Topic>.vue` |
| Settings state | `composables/useDesignDemo.ts` |
| The floating panel | `components/dev/DesignPanel.vue` |

The panel and composable templates are in [PANEL.md](PANEL.md); copy them and fill in the presets and
options. The page calls `useDesignDemo` **once** and passes `settings` down as a prop.

Rules for the presets:

- **Real data, real surroundings.** Mount the presets where the design will live, fed by the page's own
  data. If the data doesn't cover the interesting cases, seed them through the app's API or fixtures, or
  pick records that show them. Examples are a good *and* a bad outcome, an empty list, or a long name.
- **Presets disagree for real.** For UX, that means different layout and hierarchy, not recoloured copies.
  For UI, it means distinct visual languages on the same structure.
- **Every preset follows the project's design rules.** A preset that breaks a rule is judged on the wrong
  thing.
- **Knobs are cheap to honour.** A knob maps to a class, a filter or a formatter argument, never a fork of a
  component. A knob that only makes sense for some presets lists them in `presets` and the panel hides it
  elsewhere.
- **Keep each preset in its own component**, sharing only the pure helpers, so deleting the losers is
  deleting files.

Run the typecheck and the helper tests.

## 3. Verify in the browser

Run the app the way the project says to, and use the URL it prints (don't assume a port: another checkout
may hold it). Open the page with whatever browser tooling is available (a DevTools / browser MCP, or a
headless Playwright script in a scratch directory).

A Playwright script must:
- set the settings in `localStorage` (the key is in PANEL.md);
- reload;
- screenshot the scoped section, once per preset and for the knobs that change the most;
- collect console errors.

Look at every screenshot yourself before handing over, and fix what reads badly. Examples are missing
spaces, a value flagged as "changed" that always differs, or a grey that isn't secondary.

## 4. Hand over

Tell the user:
- the URL, including the query that shows the interesting case;
- where the panel is (bottom-right, dev only);
- one line per preset;
- the knobs.

Then ask them to press **Copy settings** and paste the JSON back, with notes if they have any. The panel
has a notes field, and its text is included in the JSON.

Don't commit the demo. It lives in the working tree until the choice is made.

## 5. Fold in the choice

The pasted JSON names the preset and every knob's value (see PANEL.md § Copied settings). The user may
also mix presets in words ("the banner of A, the bars of C, the table of A").

1. Keep the chosen parts as the real component(s), and bake each knob's chosen value in as the behaviour.
   Remove the knob unless the user wants it kept as a real control (e.g. a "Changes / All" toggle).
   Rewrite to production standard; don't just strip the props.
2. Delete the other presets, the wrapper, `useDesignDemo`, the `DesignPanel` and any helper only a losing
   preset used, along with its tests.
3. Replace what the chosen design supersedes, and update the e2e tests for the new structure if the project
   has them.
4. If the user's feedback states a rule that holds beyond this screen ("stop using monospace for numbers",
   "only one text colour"), apply it across the app and write it into the project's design rules (create a
   `DESIGN.md` if there are none, and ask before doing so).
5. Run the typecheck, the tests and the e2e suite. Check the result in the browser, then commit with a
   message that names what was chosen.
