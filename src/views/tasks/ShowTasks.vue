<template>
	<div class="is-max-width-desktop has-text-left" v-cy="'showTasks'">
		<h3 class="mb-2 title">
			{{ pageTitle }}
		</h3>
		<p v-if="!showAll" class="show-tasks-options">
			<datepicker-with-range @update:model-value="setDate">
				<template #trigger="{toggle}">
					<x-button @click.prevent.stop="toggle()" variant="primary" :shadow="false" class="mb-2">
						{{ $t('task.show.select') }}
					</x-button>
				</template>
			</datepicker-with-range>
			<fancycheckbox @update:model-value="setShowNulls" :model-value="showNulls" class="mr-2">
				{{ $t('task.show.noDates') }}
			</fancycheckbox>
			<fancycheckbox @update:model-value="setShowOverdue" :model-value="showOverdue">
				{{ $t('task.show.overdue') }}
			</fancycheckbox>
		</p>
		<template v-if="!loading && (!tasks || tasks.length === 0) && showNothingToDo">
			<h3 class="has-text-centered mt-6">{{ $t('task.show.noTasks') }}</h3>
			<LlamaCool class="llama-cool"/>
		</template>

		<card
			v-if="hasTasks"
			:padding="false"
			class="has-overflow"
			:has-content="false"
			:loading="loading"
		>
			<div class="p-2">
				<single-task-in-project
					v-for="t in tasks"
					:key="t.id"
					:show-project="true"
					:the-task="t"
					@taskUpdated="updateTasks"/>
			</div>
		</card>
		<div v-else :class="{ 'is-loading': loading}" class="spinner"></div>
	</div>
</template>

<script setup lang="ts">
import {computed, ref, watchEffect} from 'vue'
import {useRoute, useRouter} from 'vue-router'
import {useI18n} from 'vue-i18n'

import {formatDate} from '@/helpers/time/formatDate'
import {setTitle} from '@/helpers/setTitle'

import Fancycheckbox from '@/components/input/fancycheckbox.vue'
import SingleTaskInProject from '@/components/tasks/partials/singleTaskInProject.vue'
import DatepickerWithRange from '@/components/date/datepickerWithRange.vue'
import {DATE_RANGES} from '@/components/date/dateRanges'
import LlamaCool from '@/assets/llama-cool.svg?component'
import type {ITask} from '@/modelTypes/ITask'
import {useAuthStore} from '@/stores/auth'
import {useTaskStore} from '@/stores/tasks'
import {useProjectStore} from '@/stores/projects'

const authStore = useAuthStore()
const taskStore = useTaskStore()
const route = useRoute()
const router = useRouter()
const {t} = useI18n({useScope: 'global'})

const tasks = ref<ITask[]>([])
const showNothingToDo = ref<boolean>(false)

const projectStore = useProjectStore()

setTimeout(() => showNothingToDo.value = true, 100)

// Linting disabled because we explicitely enabled destructuring in vite's config, this will work.
// eslint-disable-next-line vue/no-setup-props-destructure
const {
	dateFrom,
	dateTo,
	showNulls = false,
	showOverdue = false,
} = defineProps<{
	dateFrom?: Date | string,
	dateTo?: Date | string,
	showNulls?: boolean,
	showOverdue?: boolean,
}>()

const showAll = computed(() => typeof dateFrom === 'undefined' || typeof dateTo === 'undefined')

const pageTitle = computed(() => {
	// We need to define "key" because it is the first parameter in the array and we need the second
	const predefinedRange = Object.entries(DATE_RANGES)
		// eslint-disable-next-line no-unused-vars
		.find(([, value]) => dateFrom === value[0] && dateTo === value[1])
		?.[0]
	if (typeof predefinedRange !== 'undefined') {
		return t(`input.datepickerRange.ranges.${predefinedRange}`)
	}

	return showAll.value
		? t('task.show.titleCurrent')
		: t('task.show.fromuntil', {
			from: formatDate(dateFrom, 'PPP'),
			until: formatDate(dateTo, 'PPP'),
		})
})
const hasTasks = computed(() => tasks.value && tasks.value.length > 0)
const userAuthenticated = computed(() => authStore.authenticated)
const loading = computed(() => taskStore.isLoading)

interface dateStrings {
	dateFrom: string,
	dateTo: string,
}

function setDate(dates: dateStrings) {
	router.push({
		name: route.name as string,
		query: {
			from: dates.dateFrom ?? dateFrom,
			to: dates.dateTo ?? dateTo,
			showOverdue: showOverdue ? 'true' : 'false',
			showNulls: showNulls ? 'true' : 'false',
		},
	})
}

function setShowOverdue(show: boolean) {
	router.push({
		name: route.name as string,
		query: {
			...route.query,
			showOverdue: show ? 'true' : 'false',
		},
	})
}

function setShowNulls(show: boolean) {
	router.push({
		name: route.name as string,
		query: {
			...route.query,
			showNulls: show ? 'true' : 'false',
		},
	})
}

async function loadPendingTasks(from: string, to: string) {
	// FIXME: HACK! This should never happen.
	// Since this route is authentication only, users would get an error message if they access the page unauthenticated.
	// Since this component is mounted as the home page before unauthenticated users get redirected
	// to the login page, they will almost always see the error message.
	if (!userAuthenticated.value) {
		return
	}

	const params = {
		sortBy: ['due_date', 'id'],
		orderBy: ['asc', 'desc'],
		filterBy: ['done'],
		filterValue: ['false'],
		filterComparator: ['equals'],
		filterConcat: 'and',
		filterIncludeNulls: showNulls,
	}

	if (!showAll.value) {
		params.filterBy.push('due_date')
		params.filterValue.push(to)
		params.filterComparator.push('less')

		// NOTE: Ideally we could also show tasks with a start or end date in the specified range, but the api
		//       is not capable (yet) of combining multiple filters with 'and' and 'or'.

		if (!showOverdue) {
			params.filterBy.push('due_date')
			params.filterValue.push(from)
			params.filterComparator.push('greater')
		}
	}
	
	if (authStore.settings.frontendSettings.filterIdUsedOnOverview && typeof projectStore.projects[authStore.settings.frontendSettings.filterIdUsedOnOverview] !== 'undefined') {
		tasks.value = await taskStore.loadTasks(params, authStore.settings.frontendSettings.filterIdUsedOnOverview)
		return
	}

	tasks.value = await taskStore.loadTasks(params)
}

// FIXME: this modification should happen in the store
function updateTasks(updatedTask: ITask) {
	for (const t in tasks.value) {
		if (tasks.value[t].id === updatedTask.id) {
			tasks.value[t] = updatedTask
			// Move the task to the end of the done tasks if it is now done
			if (updatedTask.done) {
				tasks.value.splice(t, 1)
				tasks.value.push(updatedTask)
			}
			break
		}
	}
}

watchEffect(() => loadPendingTasks(dateFrom as string, dateTo as string))
watchEffect(() => setTitle(pageTitle.value))
</script>

<style lang="scss" scoped>
.show-tasks-options {
	display: flex;
	flex-direction: column;
}

.llama-cool {
	margin: 3rem auto 0;
	display: block;
}
</style>