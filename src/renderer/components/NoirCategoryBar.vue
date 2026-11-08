<!-- NoirTube is English-only, so these labels are not translated -->
<!-- eslint-disable @intlify/vue-i18n/no-raw-text -->
<template>
  <div class="noirCategoryBar">
    <div
      class="chips"
      role="tablist"
      aria-label="Categories"
    >
      <button
        type="button"
        class="chip"
        role="tab"
        :aria-selected="selected === ''"
        :class="{ active: selected === '' }"
        @click="select('')"
      >
        All
      </button>
      <button
        v-for="(cat, i) in categories"
        :key="cat.name + i"
        type="button"
        class="chip"
        role="tab"
        :aria-selected="selected === cat.name"
        :class="{ active: selected === cat.name }"
        @click="editing ? startEdit(i) : select(cat.name)"
      >
        {{ cat.name }}
        <span
          v-if="!editing"
          class="count"
        >{{ counts[i] }}</span>
        <span
          v-else
          class="pencil"
          aria-hidden="true"
        >✎</span>
      </button>
      <button
        type="button"
        class="chip ghost"
        @click="startEdit(-1)"
      >
        + Category
      </button>
      <button
        v-if="categories.length > 0"
        type="button"
        class="chip ghost"
        :class="{ active: editing }"
        @click="editing = !editing; form = null"
      >
        {{ editing ? 'Done' : 'Edit' }}
      </button>
    </div>

    <form
      v-if="form"
      class="editor"
      @submit.prevent="save"
    >
      <label>
        Name
        <input
          v-model="form.name"
          type="text"
          maxlength="30"
          placeholder="Rivals"
        >
      </label>
      <label>
        Words to match in the title or channel name, separated by commas
        <input
          v-model="form.keywords"
          type="text"
          placeholder="rivals, rivals roblox"
        >
      </label>
      <div class="actions">
        <button
          type="submit"
          class="chip active"
          :disabled="!form.name.trim() || !form.keywords.trim()"
        >
          Save
        </button>
        <button
          type="button"
          class="chip"
          @click="form = null"
        >
          Cancel
        </button>
        <button
          v-if="form.index >= 0"
          type="button"
          class="chip danger"
          @click="remove(form.index)"
        >
          Delete
        </button>
      </div>
    </form>
  </div>
</template>

<script setup>
import { computed, ref, watch } from 'vue'

const props = defineProps({
  videoList: {
    type: Array,
    required: true
  }
})

const emit = defineEmits(['change'])

const STORAGE_KEY = 'noirtube.categories'
const SELECTED_KEY = 'noirtube.selectedCategory'
const DEFAULTS = [
  { name: 'Rivals', keywords: ['rivals'] }
]

function load() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (raw) {
      const parsed = JSON.parse(raw)
      if (Array.isArray(parsed)) {
        return parsed.filter((c) => c && typeof c.name === 'string' && Array.isArray(c.keywords))
      }
    }
  } catch { }
  return DEFAULTS.map((c) => ({ ...c, keywords: [...c.keywords] }))
}

const categories = ref(load())
let initialSelected = ''
try { initialSelected = localStorage.getItem(SELECTED_KEY) || '' } catch { }
const selected = ref(categories.value.some((c) => c.name === initialSelected) ? initialSelected : '')
const editing = ref(false)
const form = ref(null)

function persist() {
  try { localStorage.setItem(STORAGE_KEY, JSON.stringify(categories.value)) } catch { }
}

function matches(video, cat) {
  const text = `${video.title || ''} ${video.author || ''}`.toLowerCase()
  return cat.keywords.some((k) => k && text.includes(k.toLowerCase()))
}

const counts = computed(() => categories.value.map((cat) => props.videoList.filter((v) => matches(v, cat)).length))

const filtered = computed(() => {
  const cat = categories.value.find((c) => c.name === selected.value)
  return cat ? props.videoList.filter((v) => matches(v, cat)) : props.videoList
})

watch(filtered, (list) => emit('change', list), { immediate: true })

function select(name) {
  selected.value = name
  try { localStorage.setItem(SELECTED_KEY, name) } catch { }
}

function startEdit(index) {
  const cat = categories.value[index]
  form.value = cat
    ? { index, name: cat.name, keywords: cat.keywords.join(', ') }
    : { index: -1, name: '', keywords: '' }
}

function save() {
  const entry = {
    name: form.value.name.trim(),
    keywords: form.value.keywords.split(',').map((k) => k.trim()).filter(Boolean)
  }
  if (!entry.name || entry.keywords.length === 0) { return }
  const list = [...categories.value]
  if (form.value.index >= 0) {
    const old = list[form.value.index].name
    list[form.value.index] = entry
    if (selected.value === old) { select(entry.name) }
  } else {
    list.push(entry)
  }
  categories.value = list
  persist()
  form.value = null
}

function remove(index) {
  const [gone] = categories.value.splice(index, 1)
  categories.value = [...categories.value]
  if (gone && selected.value === gone.name) { select('') }
  persist()
  form.value = null
}
</script>

<style scoped>
.noirCategoryBar {
  margin-block: 0 16px;
}

.chips {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.chip {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 7px 14px;
  border: 1px solid var(--tertiary-text-color);
  border-radius: 999px;
  background: transparent;
  color: var(--primary-text-color);
  font: inherit;
  font-size: 14px;
  font-weight: 600;
  letter-spacing: 0.02em;
  cursor: pointer;
  transition: background-color 0.15s, color 0.15s;
}

.chip:hover {
  background: var(--side-nav-hover-color);
}

.chip.active {
  background: var(--primary-text-color);
  color: var(--bg-color);
  border-color: var(--primary-text-color);
}

.chip.ghost {
  border-style: dashed;
  color: var(--secondary-text-color);
}

.chip.danger {
  margin-inline-start: auto;
}

.chip:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.count {
  font-size: 12px;
  opacity: 0.6;
}

.editor {
  display: grid;
  gap: 12px;
  margin-block-start: 14px;
  padding: 14px;
  border: 1px solid var(--tertiary-text-color);
  border-radius: 10px;
}

.editor label {
  display: grid;
  gap: 6px;
  font-size: 13px;
  color: var(--secondary-text-color);
}

.editor input {
  padding: 9px 12px;
  border: 1px solid var(--tertiary-text-color);
  border-radius: 8px;
  background: var(--bg-color);
  color: var(--primary-text-color);
  font: inherit;
  font-size: 15px;
}

.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
</style>
