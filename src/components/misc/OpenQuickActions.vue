<script setup lang="ts">
import BaseButton from '@/components/base/BaseButton.vue'
import {useBaseStore} from '@/stores/base'
import {onBeforeUnmount, onMounted} from 'vue'
import {eventToHotkeyString} from '@github/hotkey'

const baseStore = useBaseStore()
const GLOBAL_HOTKEY = 'Control+k'

// See https://github.com/github/hotkey/discussions/85#discussioncomment-5214660
function openQuickActionsViaHotkey(event) {
	const hotkeyString = eventToHotkeyString(event)
	if (!hotkeyString) return
	if (hotkeyString !== GLOBAL_HOTKEY) return
	event.preventDefault()

	openQuickActions()
}

onMounted(() => {
	document.addEventListener('keydown', openQuickActionsViaHotkey)
})

onBeforeUnmount(() => {
	document.removeEventListener('keydown', openQuickActionsViaHotkey)
})

function openQuickActions() {
	baseStore.setQuickActionsActive(true)
}
</script>

<template>
	<BaseButton
		@click="openQuickActions"
		class="trigger-button"
		:title="$t('keyboardShortcuts.quickSearch')"
	>
		<icon icon="search"/>
	</BaseButton>
</template>
