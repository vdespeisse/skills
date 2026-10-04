# Panel and settings templates

Copy these into the app, then fill in the presets and options for the topic. Both files are deleted
when the choice is folded in. Paths are relative to the Nuxt source dir (`app/` in Nuxt 4); in a plain
Vue + Vite app put them under `src/` and see § Without Nuxt.

## `composables/useDesignDemo.ts`

```ts
// Dev only (a /ui-design demo): the presets and options of one design question, the choice kept in this
// browser between reloads, and the settings as the JSON the user copies back. Deleted once they choose.

export interface DesignPreset {
  readonly value: string
  readonly label: string
}

export type DesignOption =
  | { readonly kind: 'choice'; readonly key: string; readonly label: string; readonly choices: readonly DesignPreset[]; readonly presets?: readonly string[] }
  | { readonly kind: 'toggle'; readonly key: string; readonly label: string; readonly presets?: readonly string[] }

export interface DesignDemo {
  // What the demo answers, and where: copied with the settings so the pasted JSON stands on its own.
  readonly topic: string
  readonly question: string
  readonly presets: readonly DesignPreset[]
  readonly options: readonly DesignOption[]
}

export interface DesignSettings {
  preset: string
  options: Record<string, string | boolean>
  notes: string
}

// A toggle starts off, a choice on its first value.
function defaultOf(option: DesignOption): string | boolean {
  if (option.kind === 'toggle') return false
  return option.choices[0]?.value ?? ''
}

const defaultsOf = (demo: DesignDemo): DesignSettings => ({
  preset: demo.presets[0]?.value ?? '',
  options: Object.fromEntries(demo.options.map(option => [option.key, defaultOf(option)])),
  notes: '',
})

const storageKey = (demo: DesignDemo) => `design-demo:${demo.topic}`

function stored(demo: DesignDemo): Partial<DesignSettings> {
  try {
    const raw = localStorage.getItem(storageKey(demo))
    if (raw === null) return {}
    return JSON.parse(raw) as Partial<DesignSettings>
  } catch {
    return {}
  }
}

// Call once, from the page; pass `settings` down.
export function useDesignDemo(demo: DesignDemo, overrides: DesignSettings['options'] = {}) {
  const defaults = defaultsOf(demo)
  defaults.options = { ...defaults.options, ...overrides }
  const settings = useState<DesignSettings>(storageKey(demo), () => structuredClone(defaults))
  onMounted(() => {
    const kept = stored(demo)
    settings.value = { ...defaults, ...kept, options: { ...defaults.options, ...kept.options } }
  })
  watch(
    settings,
    value => {
      try {
        localStorage.setItem(storageKey(demo), JSON.stringify(value))
      } catch {
        // A convenience only.
      }
    },
    { deep: true }
  )
  return settings
}

// The settings the user copies back: every value with its label, so the JSON reads without the code.
export function copiedSettings(demo: DesignDemo, settings: DesignSettings, page: string) {
  const labelOf = (choices: readonly DesignPreset[], value: unknown) =>
    choices.find(choice => choice.value === value)?.label ?? String(value)
  return {
    topic: demo.topic,
    question: demo.question,
    page,
    preset: { value: settings.preset, label: labelOf(demo.presets, settings.preset) },
    options: Object.fromEntries(
      demo.options
        .filter(option => option.presets === undefined || option.presets.includes(settings.preset))
        .map(option => {
          const value = settings.options[option.key]
          if (option.kind === 'toggle') return [option.key, { label: option.label, value }]
          return [option.key, { label: option.label, value, choice: labelOf(option.choices, value) }]
        })
    ),
    notes: settings.notes,
  }
}
```

## `components/dev/DesignPanel.vue`

```vue
<script setup lang="ts">
import { type DesignDemo, type DesignSettings, copiedSettings } from '~/composables/useDesignDemo'

// Dev only (a /ui-design demo): switches the presets, sets their options, takes notes, and copies the
// settings as JSON for the user to paste back. Floats bottom-right and folds to one button.
const { demo } = defineProps<{ demo: DesignDemo }>()
const settings = defineModel<DesignSettings>({ required: true })

const open = ref(true)
const route = useRoute()
const toast = useToast()

const shownOptions = computed(() =>
  demo.options.filter(option => option.presets === undefined || option.presets.includes(settings.value.preset))
)

function set(key: string, value: string | boolean) {
  settings.value = { ...settings.value, options: { ...settings.value.options, [key]: value } }
}

async function copy() {
  const json = JSON.stringify(copiedSettings(demo, settings.value, route.fullPath), null, 2)
  try {
    await navigator.clipboard.writeText(json)
    toast.add({ title: 'Settings copied', description: 'Paste them back to the agent.', color: 'success' })
  } catch {
    // No clipboard (an insecure origin): show the JSON to copy by hand.
    window.prompt('Copy the settings', json)
  }
}
</script>

<template>
  <div class="fixed right-4 bottom-4 z-50 flex flex-col items-end gap-2">
    <UCard v-if="open" class="max-h-[80dvh] w-80 overflow-y-auto shadow-lg" :ui="{ body: 'flex flex-col gap-4 text-sm' }">
      <div class="flex items-center justify-between gap-2">
        <p class="font-semibold">{{ demo.question }}</p>
        <UBadge color="warning" variant="subtle">dev</UBadge>
      </div>
      <URadioGroup v-model="settings.preset" :items="[...demo.presets]" legend="Preset" />
      <template v-for="option in shownOptions" :key="option.key">
        <USwitch
          v-if="option.kind === 'toggle'"
          :model-value="settings.options[option.key] === true"
          :label="option.label"
          @update:model-value="value => set(option.key, value)"
        />
        <URadioGroup
          v-else
          :model-value="String(settings.options[option.key])"
          @update:model-value="value => set(option.key, String(value))"
          :items="[...option.choices]"
          :legend="option.label"
          orientation="horizontal"
        />
      </template>
      <UTextarea v-model="settings.notes" placeholder="Notes for the agent (optional)" :rows="3" autoresize />
      <UButton icon="i-lucide-clipboard-copy" label="Copy settings" block @click="copy" />
    </UCard>
    <UButton
      :icon="open ? 'i-lucide-chevron-down' : 'i-lucide-palette'"
      color="neutral"
      :label="open ? 'Hide' : 'Design'"
      @click="open = !open"
    />
  </div>
</template>
```

## Wiring it on the page

```vue
<script setup lang="ts">
// Dev only: the /ui-design demo of <topic>; deleted once a preset is chosen.
const SUMMARY_DEMO: DesignDemo = {
  topic: 'compare-summary',
  question: 'Compare summary',
  presets: [
    { value: 'banner', label: 'A · Verdict banner + diff table' },
    { value: 'scoreboard', label: 'B · Scoreboard + git-style diff' },
  ],
  options: [
    { kind: 'choice', key: 'tone', label: 'Verdict frame', choices: [{ value: 'tinted', label: 'Tinted' }, { value: 'outline', label: 'Outline' }] },
    { kind: 'toggle', key: 'onlyDifferences', label: 'Diff: only what differs', presets: ['banner'] },
  ],
}
const design = useDesignDemo(SUMMARY_DEMO)
const showDesign = import.meta.dev
</script>

<template>
  <!-- … the page … -->
  <ComparisonSummaryDemo :comparison="compared" :settings="design" />
  <DevDesignPanel v-if="showDesign" v-model="design" :demo="SUMMARY_DEMO" />
</template>
```

The wrapper switches on `settings.preset`. Each preset reads the knobs it cares about from
`settings.options`. `useDesignDemo`'s second argument overrides a default: pass the value a knob should
start at when it isn't the first choice, or when a toggle should start on.

## Without Nuxt

The templates lean on Nuxt auto-imports. In a plain Vue + Vite app with Nuxt UI:

- Import `ref`, `computed`, `watch` and `onMounted` from `vue`, `useRoute` from `vue-router`, `useToast`
  from `@nuxt/ui/composables` (or wherever the project's Nuxt UI version exports it), and the components
  and composable explicitly if the project doesn't auto-import them.
- Replace `useState(key, init)` with a module-level `ref(init())` keyed by topic (a `Map` of refs is
  enough). It only has to survive between components on one page.
- Gate the panel on `import.meta.env.DEV` instead of `import.meta.dev`.
- Rename `<DevDesignPanel>` to however the project names the imported component.

## Copied settings

What **Copy settings** puts on the clipboard, and what the user pastes back:

```json
{
  "topic": "compare-summary",
  "question": "Compare summary",
  "page": "/compare?baseline=junior&candidate=senior",
  "preset": { "value": "banner", "label": "A · Verdict banner + diff table" },
  "options": {
    "tone": { "label": "Verdict frame", "value": "outline", "choice": "Outline" },
    "onlyDifferences": { "label": "Diff: only what differs", "value": true }
  },
  "notes": "Keep the bars of C under the banner."
}
```

Only the options that apply to the chosen preset are copied. Fold the choice in as SKILL.md § 5 says: the
preset is the design, each option's value becomes its fixed behaviour, and the notes can override both.
