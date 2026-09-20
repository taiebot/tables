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
			<h3 class="context-resource-cards__title">
				<span v-if="resource.emoji">{{ resource.emoji }}&nbsp;</span>{{ resource.title }}
			</h3>
			<p v-if="plainDescription(resource.description)" class="context-resource-cards__description">
				{{ plainDescription(resource.description) }}
			</p>
		</button>
	</div>
</template>

<script>
import DOMPurify from 'dompurify'

export default {
	name: 'ContextResourceCards',

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
		/**
		 * @param {string} description raw, possibly rich-text, description
		 * @return {string} plain-text, whitespace-collapsed excerpt
		 */
		plainDescription(description) {
			if (!description) {
				return ''
			}
			return DOMPurify.sanitize(description, { ALLOWED_TAGS: [] }).replace(/\s+/g, ' ').trim()
		},
	},
}
</script>

<style scoped lang="scss">
.context-resource-cards {
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
		min-height: calc(30 * var(--default-grid-baseline, 4px));
		text-align: start;
		cursor: pointer;
		user-select: none;
		background-color: var(--color-main-background);
		border: 2px solid var(--color-border);
		border-radius: var(--border-radius-large, 12px);
		padding: calc(5 * var(--default-grid-baseline, 4px));
		transition: border-color var(--animation-quick, 100ms) ease, background-color var(--animation-quick, 100ms) ease;

		&:hover {
			border-color: var(--color-primary-element);
		}

		&:focus {
			outline: none;
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
		margin: 0 0 calc(2 * var(--default-grid-baseline, 4px));
		overflow-wrap: break-word;
	}

	&__description {
		margin: 0;
		color: var(--color-text-maxcontrast);
		display: -webkit-box;
		-webkit-line-clamp: 3;
		-webkit-box-orient: vertical;
		overflow: hidden;
	}
}
</style>
