<!--
  - SPDX-FileCopyrightText: 2026 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div class="context-resource-cards" role="tablist" :aria-label="t('tables', 'Application resources')">
		<button v-for="(resource, index) in resources"
			:id="`context-resource-card-${resource.key}`"
			:key="resource.key"
			type="button"
			role="tab"
			class="context-resource-cards__card"
			:class="{ 'context-resource-cards__card--active': index === activeIndex }"
			:aria-selected="index === activeIndex"
			@click="$emit('update:active-index', index)">
			<div class="context-resource-cards__title">
				<span v-if="resource.emoji">{{ resource.emoji }}&nbsp;</span>{{ resource.title }}
			</div>
			<div v-if="plainDescription(resource.description)" class="context-resource-cards__description">
				{{ plainDescription(resource.description) }}
			</div>
		</button>
	</div>
</template>

<script>
import DOMPurify from 'dompurify'

export default {
	name: 'ContextResourceCards',

	props: {
		/**
		 * The list of resources (tables/views) shown on the current context page.
		 * Each entry is expected to expose `key`, `title`, `emoji` and `description`,
		 * matching the objects already pushed into `contextResources` by Context.vue.
		 */
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
		/**
		 * Resource descriptions are stored as rich text (HTML). For the card preview we
		 * only want a short plain-text excerpt, so strip all tags via DOMPurify rather
		 * than rendering (and having to sanitize) HTML inside the card.
		 *
		 * @param {string} description the raw, possibly empty, rich-text description
		 * @return {string} a plain-text, whitespace-collapsed excerpt
		 */
		plainDescription(description) {
			if (!description) {
				return ''
			}
			const text = DOMPurify.sanitize(description, { ALLOWED_TAGS: [] }).replace(/\s+/g, ' ').trim()
			return text
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

	&__card {
		appearance: none;
		box-sizing: border-box;
		width: 100%;
		text-align: start;
		cursor: pointer;
		background-color: var(--color-main-background);
		border: 2px solid var(--color-border);
		border-radius: var(--border-radius-large, 12px);
		padding: calc(3 * var(--default-grid-baseline, 4px));
		transition: border-color var(--animation-quick, 100ms) ease, background-color var(--animation-quick, 100ms) ease;

		&:hover,
		&:focus-visible {
			border-color: var(--color-primary-element);
		}

		&:focus-visible {
			outline: 2px solid var(--color-primary-element);
			outline-offset: 2px;
		}

		&--active {
			border-color: var(--color-primary-element);
			background-color: var(--color-primary-element-light);
		}
	}

	&__title {
		font-weight: bold;
		font-size: 15px;
		margin-bottom: calc(1 * var(--default-grid-baseline, 4px));
		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
	}

	&__description {
		font-size: 12px;
		color: var(--color-text-maxcontrast);
		line-height: 1.3;
		display: -webkit-box;
		-webkit-line-clamp: 2;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}
}
</style>
