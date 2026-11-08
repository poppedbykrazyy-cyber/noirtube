<!-- NoirTube is English-only, so these labels are not translated -->
<!-- eslint-disable @intlify/vue-i18n/no-raw-text -->
<template>
  <div class="noirCopySubs">
    <h3>Copy my subscriptions</h3>
    <ol>
      <li>
        <button
          type="button"
          class="pill"
          @click="copyScript"
        >
          Copy the script
        </button>
      </li>
      <li>
        Open
        <button
          type="button"
          class="link"
          @click="openChannelsPage"
        >
          youtube.com/feed/channels
        </button>
        in your normal browser, signed in to YouTube.
      </li>
      <li>
        Press <b>F12</b>, click <b>Console</b>, paste with <b>Ctrl+V</b> and press <b>Enter</b>.
        If Chrome asks, type <b>allow pasting</b> first.
        Wait until it says it copied your channels.
      </li>
      <li>Come back here, click the box below, press <b>Ctrl+V</b>, then <b>Import</b>.</li>
    </ol>
    <textarea
      v-model="pasted"
      rows="3"
      placeholder="Paste here"
      aria-label="Paste your subscriptions here"
    />
    <div class="row">
      <button
        type="button"
        class="pill solid"
        :disabled="busy || !pasted.trim()"
        @click="importPasted"
      >
        {{ busy ? `Importing… ${progress}` : 'Import' }}
      </button>
      <span
        v-if="result"
        class="result"
      >{{ result }}</span>
    </div>
  </div>
</template>

<script setup>
import { computed, ref } from 'vue'

import store from '../store/index'
import { copyToClipboard, deepCopy, openExternalLink, showToast } from '../helpers/utils'
import { getLocalChannelId } from '../helpers/api/local'

// Runs on youtube.com/feed/channels in the user's own browser. It scrolls the list so every
// channel loads, then copies id, name, link and picture for each one. Nothing is sent anywhere.
const SCRIPT = `(async () => {
  let last = -1
  for (let i = 0; i < 60; i++) {
    window.scrollTo(0, document.documentElement.scrollHeight)
    await new Promise((r) => setTimeout(r, 1200))
    const n = document.querySelectorAll('ytd-channel-renderer').length
    if (n === last) break
    last = n
  }
  const list = [...document.querySelectorAll('ytd-channel-renderer')].map((el) => {
    const d = el.data || (el.__data && el.__data.data) || {}
    const a = el.querySelector('a#main-link') || el.querySelector('a[href*="/@"]') || el.querySelector('a[href*="/channel/"]')
    const name = (d.title && d.title.simpleText) || (el.querySelector('#channel-title #text, #text') || {}).textContent || ''
    const thumb = (d.thumbnail && d.thumbnail.thumbnails && d.thumbnail.thumbnails[0] && d.thumbnail.thumbnails[0].url) || ''
    return { id: d.channelId || '', name: name.trim(), url: a ? a.href : '', thumbnail: thumb.startsWith('//') ? 'https:' + thumb : thumb }
  }).filter((c) => c.id || c.url)
  const text = 'NOIRTUBE' + JSON.stringify(list)
  let copied = false
  try { copy(text); copied = true } catch (e) {}
  if (!copied) { try { await navigator.clipboard.writeText(text); copied = true } catch (e) {} }
  if (!copied) {
    const box = document.createElement('textarea')
    box.value = text
    box.style.cssText = 'position:fixed;inset:20px;z-index:99999;font-size:12px'
    document.body.appendChild(box)
    box.select()
    alert('NoirTube: press Ctrl+C now to copy ' + list.length + ' channels, then paste them into NoirTube.')
    return
  }
  alert('NoirTube: copied ' + list.length + ' channels. Now paste them into NoirTube.')
})()`

const pasted = ref('')
const busy = ref(false)
const progress = ref('')
const result = ref('')

const profileList = computed(() => store.getters.getProfileList)

function copyScript() {
  copyToClipboard(SCRIPT, { messageOnSuccess: 'Script copied. Now open your YouTube channels page.' })
}

function openChannelsPage() {
  openExternalLink('https://www.youtube.com/feed/channels')
}

function parsePasted(text) {
  const trimmed = text.trim()
  if (trimmed.startsWith('NOIRTUBE')) {
    const list = JSON.parse(trimmed.slice('NOIRTUBE'.length))
    return Array.isArray(list) ? list : []
  }
  // Also accept one channel link or UC... id per line
  return trimmed.split(/\s+/).filter(Boolean).map((line) => {
    const id = line.match(/(UC[\w-]{22})/)?.[1] || ''
    return { id, name: '', url: id ? '' : line, thumbnail: '' }
  })
}

async function importPasted() {
  let entries
  try {
    entries = parsePasted(pasted.value)
  } catch {
    result.value = 'That didn’t look right. Run the script again and paste the whole thing.'
    return
  }
  if (entries.length === 0) {
    result.value = 'Nothing to import.'
    return
  }

  busy.value = true
  result.value = ''
  const profile = deepCopy(profileList.value[0])
  const have = new Set(profile.subscriptions.map((c) => c.id))
  const added = []
  let failed = 0

  // Channels copied as /@handle links need one lookup each to get their id
  let done = 0
  const queue = [...entries]
  async function worker() {
    while (queue.length) {
      const entry = queue.shift()
      let id = entry.id
      if (!id && entry.url && process.env.SUPPORTS_LOCAL_API) {
        try { id = await getLocalChannelId(entry.url) } catch { }
      }
      if (id && /^UC[\w-]{22}$/.test(id)) {
        if (!have.has(id)) {
          have.add(id)
          added.push({ id, name: entry.name || entry.url?.split('/@')[1] || id, thumbnail: entry.thumbnail || '' })
        }
      } else {
        failed++
      }
      done++
      progress.value = `${done}/${entries.length}`
    }
  }
  await Promise.all([worker(), worker(), worker(), worker()])

  if (added.length > 0) {
    profile.subscriptions = profile.subscriptions.concat(added)
    await store.dispatch('updateProfile', profile)
  }
  busy.value = false
  progress.value = ''
  pasted.value = ''
  result.value = `Added ${added.length} channel${added.length === 1 ? '' : 's'}` +
    (failed ? `, ${failed} couldn’t be found.` : '.')
  if (added.length > 0) {
    showToast(`NoirTube: added ${added.length} channels. Your feed will load them now.`)
  }
}
</script>

<style scoped>
.noirCopySubs {
  line-height: 1.5;
}

.noirCopySubs h3 {
  margin-block: 0 8px;
}

.noirCopySubs ol {
  padding-inline-start: 20px;
}

.noirCopySubs li {
  margin-block-end: 6px;
}

textarea {
  box-sizing: border-box;
  inline-size: 100%;
  padding-block: 8px;
  padding-inline: 10px;
  border: 1px solid var(--tertiary-text-color);
  border-radius: 8px;
  background: var(--bg-color);
  color: var(--primary-text-color);
  font: inherit;
  font-size: 13px;
  resize: vertical;
}

.row {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 12px;
  margin-block-start: 10px;
}

.pill {
  padding-block: 6px;
  padding-inline: 16px;
  border: 1px solid var(--primary-text-color);
  border-radius: 999px;
  background: transparent;
  color: var(--primary-text-color);
  font: inherit;
  font-weight: 700;
  cursor: pointer;
}

.pill.solid {
  background: var(--primary-text-color);
  color: var(--bg-color);
}

.pill:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.link {
  padding: 0;
  border: 0;
  background: none;
  color: var(--primary-text-color);
  font: inherit;
  font-weight: 700;
  text-decoration: underline;
  cursor: pointer;
}

.result {
  color: var(--secondary-text-color);
}
</style>
