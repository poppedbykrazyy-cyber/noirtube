<!-- NoirTube is English-only, so these labels are not translated -->
<!-- eslint-disable @intlify/vue-i18n/no-raw-text -->
<template>
  <div>
    <FtCard class="card">
      <div class="header">
        <h2>
          <FontAwesomeIcon
            :icon="['fas', 'fire']"
            class="headerIcon"
          />
          For You
        </h2>
        <FtButton
          label="Refresh"
          :disabled="isLoading"
          background-color="var(--primary-color)"
          text-color="var(--text-with-main-color)"
          @click="build(true)"
        />
      </div>
      <p class="hint">
        Picked from your subscriptions and what you watch in NoirTube. The more you watch, the better it gets.
      </p>

      <FtLoader v-if="isLoading" />
      <template v-else>
        <div
          v-if="videos.length === 0"
          class="empty"
        >
          <p v-if="!hasSubscriptions && watchedCount === 0">
            Subscribe to some channels or watch a few videos, and your For You page will fill up.
          </p>
          <p v-else>
            Couldn't load suggestions right now. Check your connection and press Refresh.
          </p>
          <router-link
            to="/subscriptions"
            class="emptyLink"
          >
            Go to Subscriptions
          </router-link>
        </div>
        <template v-else>
          <NoirCategoryBar
            :video-list="videos"
            @change="shownVideos = $event"
          />
          <FtElementList
            :data="shownVideos"
            :use-channels-hidden-preference="true"
          />
        </template>
      </template>
    </FtCard>
  </div>
</template>

<script setup>
import { computed, onMounted, ref, shallowRef } from 'vue'

import FtButton from '../../components/FtButton/FtButton.vue'
import FtCard from '../../components/ft-card/ft-card.vue'
import FtElementList from '../../components/FtElementList/FtElementList.vue'
import FtLoader from '../../components/FtLoader/FtLoader.vue'
import NoirCategoryBar from '../../components/NoirCategoryBar.vue'

import store from '../../store/index'
import { getLocalRelatedVideos } from '../../helpers/api/local'
import { invidiousGetVideoInformation } from '../../helpers/api/invidious'

// Results are kept for this window, so going back to the page doesn't reload everything
const memory = window.noirForYouMemory || (window.noirForYouMemory = { videos: null })

const isLoading = ref(true)
const videos = shallowRef([])
const shownVideos = shallowRef([])

const activeProfile = computed(() => store.getters.getActiveProfile)
const hasSubscriptions = computed(() => activeProfile.value.subscriptions.length > 0)
const history = computed(() => store.getters.getHistoryCacheSorted)
const watchedCount = computed(() => history.value.length)
const backendPreference = computed(() => store.getters.getBackendPreference)
const backendFallback = computed(() => store.getters.getBackendFallback)

const STOP_WORDS = new Set(('the and for you your with this that from what have just are was but not all out new how why who' +
  ' its his her they them our its into than then when will can get got one two vs ft feat official video videos' +
  ' episode part live full best day').split(' '))

function words(text) {
  return (text || '').toLowerCase().match(/[a-z0-9]{3,}/g)?.filter((w) => !STOP_WORDS.has(w)) ?? []
}

function subscriptionVideos() {
  const cache = store.getters.getVideoCache
  const list = []
  for (const channel of activeProfile.value.subscriptions) {
    const entry = cache[channel.id]
    if (entry?.videos) { list.push(...entry.videos) }
  }
  return list.filter((v) => v.type === 'video' && v.videoId)
}

async function related(videoId) {
  const viaInvidious = async () => {
    const info = await invidiousGetVideoInformation(videoId)
    return (info.recommendedVideos || []).map((v) => ({
      type: 'video',
      videoId: v.videoId,
      title: v.title,
      author: v.author,
      authorId: v.authorId,
      viewCount: v.viewCount,
      lengthSeconds: v.lengthSeconds
    }))
  }
  const useInvidious = !process.env.SUPPORTS_LOCAL_API || backendPreference.value === 'invidious'
  try {
    return useInvidious ? await viaInvidious() : await getLocalRelatedVideos(videoId)
  } catch (error) {
    console.warn('For You: related videos failed', videoId, error)
    if (!useInvidious && backendFallback.value) {
      try { return await viaInvidious() } catch { }
    }
    return []
  }
}

function pickSeeds(subs) {
  const seeds = []
  const seen = new Set()
  const add = (id) => { if (id && !seen.has(id)) { seen.add(id); seeds.push(id) } }

  // What you watched most recently says the most about what you like right now
  history.value.slice(0, 6).forEach((h) => add(h.videoId))

  // Plus a few fresh uploads from channels you follow, so it isn't only past watches
  const fresh = [...subs].sort((a, b) => (b.published || 0) - (a.published || 0)).slice(0, 12)
  for (let i = fresh.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [fresh[i], fresh[j]] = [fresh[j], fresh[i]]
  }
  fresh.slice(0, 10 - Math.min(seeds.length, 6)).forEach((v) => add(v.videoId))
  return seeds.slice(0, 10)
}

async function build(force = false) {
  if (!force && memory.videos) {
    videos.value = memory.videos
    isLoading.value = false
    return
  }
  isLoading.value = true

  const watched = store.getters.getHistoryCacheById
  const subscribedIds = new Set(activeProfile.value.subscriptions.map((c) => c.id))
  const subs = subscriptionVideos()

  // Words you watch a lot, like "rivals", get a boost
  const taste = new Map()
  history.value.slice(0, 200).forEach((h, i) => {
    const weight = i < 30 ? 2 : 1
    for (const w of new Set(words(h.title))) { taste.set(w, (taste.get(w) || 0) + weight) }
  })
  const tasteScore = (title) => {
    let score = 0
    for (const w of new Set(words(title))) { score += Math.min(taste.get(w) || 0, 6) }
    return Math.min(score, 15)
  }

  const seeds = pickSeeds(subs)
  const results = await Promise.all(seeds.map((id) => related(id)))

  const candidates = new Map()
  const consider = (video, base) => {
    if (!video?.videoId || watched[video.videoId] || video.isUpcoming) { return }
    const existing = candidates.get(video.videoId)
    if (existing) {
      // Suggested next to several of your videos: probably a good pick
      existing.score += 2
      return
    }
    candidates.set(video.videoId, { video, score: base })
  }

  results.forEach((list) => list.slice(0, 15).forEach((v, rank) => consider(v, 3 - rank * 0.12)))

  const weekAgo = Date.now() - 7 * 24 * 3600e3
  subs.forEach((v) => {
    if ((v.published || 0) > weekAgo) { consider(v, 3) }
  })

  for (const entry of candidates.values()) {
    const v = entry.video
    entry.score += tasteScore(v.title) * 0.6
    if (subscribedIds.has(v.authorId)) { entry.score += 1.5 }
    entry.score += Math.random() * 2
  }

  // Don't let one channel take over the page
  const perChannel = new Map()
  const ranked = [...candidates.values()]
    .sort((a, b) => b.score - a.score)
    .filter(({ video }) => {
      const n = (perChannel.get(video.authorId) || 0) + 1
      perChannel.set(video.authorId, n)
      return n <= 3
    })
    .slice(0, 72)
    .map((e) => e.video)

  memory.videos = ranked
  videos.value = ranked
  isLoading.value = false
}

onMounted(() => {
  build()
})
</script>

<style scoped>
.card {
  inline-size: 85%;
  margin-block: 0 60px;
  margin-inline: auto;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.headerIcon {
  margin-inline-end: 6px;
}

.hint {
  margin-block: 0 16px;
  color: var(--secondary-text-color);
}

.empty {
  padding-block: 20px;
  text-align: center;
}

.emptyLink {
  color: var(--primary-text-color);
  font-weight: 700;
}

@media only screen and (width <= 680px) {
  .card {
    inline-size: 90%;
  }
}
</style>
