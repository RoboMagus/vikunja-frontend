<template>
	<Teleport to="body">
		<!-- FIXME: transition should not be included in the modal -->
		<CustomTransition :name="transitionName" appear>
			<section
				v-if="enabled"
				class="modal-mask"
				:class="[
					{ 'has-overflow': overflow },
					variant,
				]"
				ref="modal"
				v-bind="attrs"
			>
				<div
					class="modal-container"
					@click.self.prevent.stop="$emit('close')"
					v-shortcut="'Escape'"
				>
					<div
						class="modal-content"
						:class="{
						'has-overflow': overflow,
						'is-wide': wide
					}"
					>
						<BaseButton
							@click="$emit('close')"
							class="close"
						>
							<icon icon="times"/>
						</BaseButton>

						<slot>
							<div class="header">
								<slot name="header"></slot>
							</div>
							<div class="content">
								<slot name="text"></slot>
							</div>
							<div class="actions">
								<x-button
									@click="$emit('close')"
									variant="tertiary"
									class="has-text-danger"
								>
									{{ $t('misc.cancel') }}
								</x-button>
								<x-button
									@click="$emit('submit')"
									variant="primary"
									v-cy="'modalPrimary'"
									:shadow="false"
								>
									{{ $t('misc.doit') }}
								</x-button>
							</div>
						</slot>
					</div>
				</div>
			</section>
		</CustomTransition>
	</Teleport>
</template>

<script lang="ts">
export default {
	inheritAttrs: false,
}
</script>

<script lang="ts" setup>
import CustomTransition from '@/components/misc/CustomTransition.vue'
import BaseButton from '@/components/base/BaseButton.vue'
import {ref, useAttrs, watchEffect} from 'vue'
import {useScrollLock} from '@vueuse/core'

const {
	enabled = true,
	overflow,
	wide,
	transitionName = 'modal',
	variant = 'default',
} = defineProps<{
	enabled?: boolean,
	overflow?: boolean,
	wide?: boolean,
	transitionName?: 'modal' | 'fade',
	variant?: 'default' | 'hint-modal' | 'scrolling',
}>()

defineEmits(['close', 'submit'])

const attrs = useAttrs()

const modal = ref<HTMLElement | null>(null)
const scrollLock = useScrollLock(modal)

watchEffect(() => {
	scrollLock.value = enabled
})
</script>

<style lang="scss" scoped>
$modal-margin: 4rem;
$modal-width: 1024px;

.modal-mask {
	position: fixed;
	z-index: 4000;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background-color: rgba(0, 0, 0, .8);
	transition: opacity 150ms ease;
	color: #fff;
}

.modal-container {
	transition: all 150ms ease;
	position: relative;
	width: 100%;
	height: 100%;
	max-height: 100vh;
	overflow: auto;
}

.default .modal-content,
.hint-modal .modal-content {
	text-align: center;
	position: absolute;
	top: 50%;
	left: 50%;
	transform: translate(-50%, -50%);

	@media screen and (max-width: $tablet) {
		margin: 0;
		top: 25%;
		transform: translate(-50%, -25%);
	}

	.header {
		font-size: 2rem;
		font-weight: 700;
	}

	.button {
		margin: 0 0.5rem;
	}
}

// scrolling-content
// used e.g. for <TaskDetailViewModal>
.scrolling .modal-content {
	max-width: $modal-width;
	width: 100%;
	margin: $modal-margin auto;

	max-height: none; // reset bulma
	overflow: visible; // reset bulma

	@media screen and (min-width: $tablet) {
		max-height: none; // reset bulma
		margin: $modal-margin auto; // reset bulma
		width: 100%;
	}

	@media screen and (max-width: $desktop) {
		margin: 0;
	}
}

.is-wide {
	max-width: $desktop;
	width: calc(100% - 2rem);
}

.hint-modal {
	z-index: 4600;

	:deep(.card-content) {
		text-align: left;

		.info {
			font-style: italic;
		}

		p {
			display: flex;
			justify-content: space-between;
			align-items: center;
		}
	}
}

.close {
	$close-button-min-space: 84px;
	$close-button-padding: 26px;
	position: fixed;
	top: 5px;
	right: $close-button-padding;
	color: var(--grey-900);
	font-size: 2rem;

	@media screen and (min-width: $desktop) and (max-width: calc(#{$desktop	} + #{$close-button-min-space})) {
		top: calc(5px + $modal-margin);
		right: 50%;
		// we align the close button to the modal until there is enough space outside for it
		transform: translateX(calc((#{$modal-width} / 2) - #{$close-button-padding}));
	}
	// we can only use light color when there is enough space for the close button next to the modal
	@media screen and (min-width: calc(#{$desktop	} + #{$close-button-min-space})) {
		color: var(--white);
	}
}
</style>

<style lang="scss">
// Close icon SVG uses currentColor, change the color to keep it visible
.dark .close {
	color: var(--grey-900);
}
</style>