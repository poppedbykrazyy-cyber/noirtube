# Instructions

- Only build for Windows (.exe). Do not make Mac or Linux builds unless the user explicitly asks for them.
- NoirTube is a FreeTube fork. NoirTube-specific code: `src/renderer/components/NoirCategoryBar.vue`, `src/renderer/components/NoirCopySubscriptions.vue`, `src/renderer/views/ForYou/ForYou.vue` (route `/foryou`, default landing page), `getLocalRelatedVideos` in `src/renderer/helpers/api/local.js`, the `body.noir` block at the end of `src/renderer/themes.css`, the import hint in `SubscriptionsTabUi`, and `_icons/*Noir*`, `_icons/noirJapan.svg`.
- Bump `version` in `package.json` for each release; the Windows workflow tags `v<version>`.
