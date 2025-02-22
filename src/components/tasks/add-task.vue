<template>
	<div class="task-add" ref="taskAdd">
		<div class="add-task__field field is-grouped">
			<p class="control has-icons-left has-icons-right is-expanded">
				<textarea
					class="add-task-textarea input"
					:class="{'textarea-empty': newTaskTitle === ''}"
					:placeholder="$t('project.list.addPlaceholder')"
					rows="1"
					v-focus
					v-model="newTaskTitle"
					ref="newTaskInput"
					@keyup="resetEmptyTitleError"
					@keydown.enter="handleEnter"
				/>
				<span class="icon is-small is-left">
					<icon icon="tasks"/>
				</span>
				<quick-add-magic :highlight-hint-icon="taskAddHovered"/>
			</p>
			<p class="control">
				<x-button
					class="add-task-button"
					:disabled="newTaskTitle === '' || loading || undefined"
					@click="addTask()"
					icon="plus"
					:loading="loading"
					:aria-label="$t('project.list.add')"
				>
					<span class="button-text">
						{{ $t('project.list.add') }}
					</span>
				</x-button>
			</p>
		</div>
		<Expandable :open="errorMessage !== ''">
			<p class="pt-3 mt-0 help is-danger" v-if="errorMessage !== ''">
				{{ errorMessage }}
			</p>
		</Expandable>
	</div>
</template>

<script setup lang="ts">
import {computed, ref} from 'vue'
import {useI18n} from 'vue-i18n'
import {useElementHover} from '@vueuse/core'

import {RELATION_KIND} from '@/types/IRelationKind'
import type {ITask} from '@/modelTypes/ITask'

import Expandable from '@/components/base/Expandable.vue'
import QuickAddMagic from '@/components/tasks/partials/quick-add-magic.vue'
import {parseSubtasksViaIndention} from '@/helpers/parseSubtasksViaIndention'
import TaskRelationService from '@/services/taskRelation'
import TaskRelationModel from '@/models/taskRelation'
import {getLabelsFromPrefix} from '@/modules/parseTaskText'

import {useAuthStore} from '@/stores/auth'
import {useTaskStore} from '@/stores/tasks'

import {useAutoHeightTextarea} from '@/composables/useAutoHeightTextarea'

const props = defineProps({
	defaultPosition: {
		type: Number,
		required: false,
	},
})

const emit = defineEmits(['taskAdded'])

const newTaskTitle = ref('')
const newTaskInput = useAutoHeightTextarea(newTaskTitle)

const {t} = useI18n({useScope: 'global'})
const authStore = useAuthStore()
const taskStore = useTaskStore()

// enable only if we don't have a modal
// onStartTyping(() => {
// 	if (newTaskInput.value === null || document.activeElement === newTaskInput.value) {
// 		return
// 	}
// 	newTaskInput.value.focus()
// })

const taskAdd = ref<HTMLTextAreaElement | null>(null)
const taskAddHovered = useElementHover(taskAdd)

const errorMessage = ref('')

function resetEmptyTitleError(e: KeyboardEvent) {
	if (
		(e.which <= 90 && e.which >= 48 || e.which >= 96 && e.which <= 105)
		&& newTaskTitle.value !== ''
	) {
		errorMessage.value = ''
	}
}

const loading = computed(() => taskStore.isLoading)

async function addTask() {
	if (newTaskTitle.value === '') {
		errorMessage.value = t('project.create.addTitleRequired')
		return
	}
	errorMessage.value = ''

	if (loading.value) {
		return
	}

	const taskTitleBackup = newTaskTitle.value
	// This allows us to find the tasks with the title they had before being parsed
	// by quick add magic.
	const createdTasks: { [key: ITask['title']]: ITask } = {}
	const tasksToCreate = parseSubtasksViaIndention(newTaskTitle.value, authStore.settings.frontendSettings.quickAddMagicMode)

	// We ensure all labels exist prior to passing them down to the create task method
	// In the store it will only ever see one task at a time so there's no way to reliably 
	// check if a new label was created before (because everything happens async).
	const allLabels = tasksToCreate.map(({title}) => getLabelsFromPrefix(title, authStore.settings.frontendSettings.quickAddMagicMode) ?? [])
	await taskStore.ensureLabelsExist(allLabels.flat())

	const newTasks = tasksToCreate.map(async ({title, project}) => {
		if (title === '') {
			return
		}

		// If the task has a project specified, make sure to use it
		let projectId = null
		if (project !== null) {
			projectId = await taskStore.findProjectId({project, projectId: 0})
		}

		const task = await taskStore.createNewTask({
			title,
			projectId: projectId || authStore.settings.defaultProjectId,
			position: props.defaultPosition,
		})
		createdTasks[title] = task
		return task
	})

	try {
		newTaskTitle.value = ''
		await Promise.all(newTasks)

		const taskRelationService = new TaskRelationService()
		const relations = tasksToCreate.map(async t => {
			const createdTask = createdTasks[t.title]
			if (typeof createdTask === 'undefined') {
				return
			}

			if (t.parent === null) {
				emit('taskAdded', createdTask)
				return
			}

			const createdParentTask = createdTasks[t.parent]
			if (typeof createdTask === 'undefined' || typeof createdParentTask === 'undefined') {
				return
			}

			const rel = await taskRelationService.create(new TaskRelationModel({
				taskId: createdTask.id,
				otherTaskId: createdParentTask.id,
				relationKind: RELATION_KIND.PARENTTASK,
			}))

			createdTask.relatedTasks[RELATION_KIND.PARENTTASK] = [createdParentTask]
			// we're only emitting here so that the relation shows up in the project
			emit('taskAdded', createdTask)

			return rel
		})
		await Promise.all(relations)
	} catch (e: any) {
		newTaskTitle.value = taskTitleBackup
		if (e?.message === 'NO_PROJECT') {
			errorMessage.value = t('project.create.addProjectRequired')
			return
		}
		throw e
	}
}

function handleEnter(e: KeyboardEvent) {
	// when pressing shift + enter we want to continue as we normally would. Otherwise, we want to create 
	// the new task(s). The vue event modifier don't allow this, hence this method.
	if (e.shiftKey) {
		return
	}

	e.preventDefault()
	addTask()
}

function focusTaskInput() {
	newTaskInput.value?.focus()
}

defineExpose({
	focusTaskInput,
})
</script>

<style lang="scss" scoped>
.task-add,
// overwrite bulma styles
.task-add .add-task__field {
	margin-bottom: 0;
}

.add-task-button {
	height: 100% !important;

	@media screen and (max-width: $mobile) {
		.button-text {
			display: none;
		}

		:deep(.icon) {
			margin: 0 !important;
		}
	}
}

.add-task-textarea {
	transition: border-color $transition;
	resize: none;
}

// Adding this class when the textarea has no text prevents the textarea from wrapping the placeholder.
.textarea-empty {
	white-space: nowrap;
	text-overflow: ellipsis;
}

.control.has-icons-left .icon,
.control.has-icons-right .icon {
	transition: all $transition;
}
</style>

<style>
button.show-helper-text {
	right: 0;
}
</style>
