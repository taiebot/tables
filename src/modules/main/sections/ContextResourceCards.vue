<!--
  - SPDX-FileCopyrightText: 2026 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div class="context-resource-cards" role="tablist" :aria-label="t('tables', 'Application resources')">
		<NcTile v-for="(resource, index) in resources"
			:key="resource.key"
			role="tab"
			:aria-selected="index === activeIndex"
			:title="cardTitle(resource)"
			:body="cardBody(resource)"
			:active="index === activeIndex"
			:tabbable="true"
			@set-template="$emit('update:active-index', index)" />
	</div>
</template>

<script>
import DOMPurify from 'dompurify'
import NcTile from '../../../shared/components/ncTile/NcTile.vue'

const TITLE_MAX_LENGTH = 60
const BODY_MAX_LENGTH = 110

export default {
	name: 'ContextResourceCards',

	components: {
		NcTile,
	},

	props: {
		resources: {
			type: Array,
			required: true,
		},
		activeIndex: {
			type: Number,
			default: 0,
		},
	},

	emits: ['update:active-index'],

	methods: {
		truncate(text, maxLength) {
			if (text.length <= maxLength) {
				return text
			}
			return text.slice(0, maxLength - 1).trim() + '…'
		},

		cardTitle(resource) {
			const title = resource.emoji ? `${resource.emoji} ${resource.title}` : resource.title
			return this.truncate(title, TITLE_MAX_LENGTH)
		},

		cardBody(resource) {
			return this.truncate(this.plainDescription(resource.description), BODY_MAX_LENGTH)
		},

		/**
		 * @param {string} description raw, possibly rich-text, description
		 * @return {string} plain, sanitized preview text
		 */
		plainDescription(description) {
			if (!description) {
				return ''
			}
			let text = DOMPurify.sanitize(description, { ALLOWED_TAGS: [] })
			// Drop markdown-style links/references that survive tag-stripping,
			// e.g. "[label](url)" -> "label", or a bare "[url]" reference -> ''.
			text = text
				.replace(/\[([^\]]*)\]\([^)]*\)/g, '$1')
				.replace(/\[https?:\/\/[^\]]*\]/g, '')
			return text.replace(/\s+/g, ' ').trim()
		},
	},
}
</script>

<style scoped lang="scss">
.context-resource-cards {
	position: sticky;
	inset-inline-start: 0;
	top: 0;
	z-index: 15;
	display: grid;
	grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
	gap: calc(3 * var(--default-grid-baseline, 4px));
	padding: calc(4 * var(--default-grid-baseline, 4px)) 20px;
	background-color: var(--color-main-background);
	border-bottom: 1px solid var(--color-border);
	width: var(--app-content-width, 100%);
}
</style>
