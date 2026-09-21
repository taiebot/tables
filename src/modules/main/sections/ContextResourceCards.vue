<!--
  - SPDX-FileCopyrightText: 2026 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<div class="context-resource-cards" role="tablist" :aria-label="t('tables', 'Application resources')">
		<div v-for="(resource, index) in resources"
			:key="resource.key"
			class="context-resource-cards__card"
			:class="{ 'context-resource-cards__card--active': index === activeIndex }">
			<button :id="`context-resource-card-${resource.key}`"
				type="button"
				role="tab"
				class="context-resource-cards__card-button"
				:aria-selected="index === activeIndex"
				@click="$emit('update:active-index', index)">
				<h3 class="context-resource-cards__title">
					<span v-if="resource.emoji">{{ resource.emoji }}&nbsp;</span>{{ resource.title }}
				</h3>
			</button>

			<button v-if="resource.description"
				type="button"
				class="context-resource-cards__toggle"
				:class="{ 'context-resource-cards__toggle--expanded': expandedIndex === index }"
				:aria-expanded="expandedIndex === index"
				:aria-label="expandedIndex === index ? t('tables', 'Hide description') : t('tables', 'Show description')"
				@click.stop="toggleExpanded(index)">
				<ChevronDown :size="18" />
			</button>

			<div v-if="expandedIndex === index && resource.description" class="context-resource-cards__preview">
				<NcRichText :text="resource.description" :autolink="true" />
			</div>
		</div>
	</div>
</template>

<script>
import { NcRichText } from '@nextcloud/vue'
import ChevronDown from 'vue-material-design-icons/ChevronDown.vue'

export default {
	name: 'ContextResourceCards',

	components: {
		NcRichText,
		ChevronDown,
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

	data() {
		return {
			expandedIndex: null,
		}
	},

	watch: {
		// Collapse any open preview when the active resource changes, so
		// switching tables doesn't leave a stale expanded card behind.
		activeIndex() {
			this.expandedIndex = null
		},
	},

	methods: {
		toggleExpanded(index) {
			this.expandedIndex = this.expandedIndex === index ? null : index
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
		position: relative;
		box-sizing: border-box;
		height: calc(30 * var(--default-grid-baseline, 4px));
		overflow-y: auto;
		background-color: var(--color-main-background);
		border: 2px solid var(--color-border);
		border-radius: var(--border-radius-large, 12px);
		padding: calc(5 * var(--default-grid-baseline, 4px));
		transition: border-color var(--animation-quick, 100ms) ease, background-color var(--animation-quick, 100ms) ease;

		&:hover {
			border-color: var(--color-primary-element);
		}

		&--active {
			border-color: var(--color-primary-element);
			background-color: var(--color-primary-element-light);
		}
	}

	&__card-button {
		appearance: none;
		box-sizing: border-box;
		width: 100%;
		text-align: start;
		cursor: pointer;
		user-select: none;
		background: transparent;
		border: none;
		padding: 0;
		margin: 0;
		font: inherit;
		color: inherit;

		&:focus {
			outline: none;
		}

		&:focus-visible {
			outline: 2px solid var(--color-primary-element);
			outline-offset: 2px;
		}
	}

	&__title {
		margin: 0;
		padding-inline-end: calc(6 * var(--default-grid-baseline, 4px));
		overflow-wrap: break-word;
	}

	&__toggle {
		appearance: none;
		position: absolute;
		top: calc(3 * var(--default-grid-baseline, 4px));
		inset-inline-end: calc(3 * var(--default-grid-baseline, 4px));
		display: flex;
		background: transparent;
		border: none;
		cursor: pointer;
		border-radius: var(--border-radius, 6px);
		padding: calc(1 * var(--default-grid-baseline, 4px));
		color: var(--color-text-maxcontrast);
		transition: transform var(--animation-quick, 100ms) ease, background-color var(--animation-quick, 100ms) ease;

		&:hover {
			color: var(--color-main-text);
			background-color: var(--color-background-hover);
		}

		&:focus {
			outline: none;
		}

		&:focus-visible {
			outline: 2px solid var(--color-primary-element);
			outline-offset: 2px;
		}

		&--expanded {
			transform: rotate(180deg);
		}
	}

	&__preview {
		margin-top: calc(2 * var(--default-grid-baseline, 4px));
		color: var(--color-text-maxcontrast);
		font-size: 14px;
	}
}
</style>
