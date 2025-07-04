<script lang="ts">
	import { FlowService, type Flow, DraftService } from '$lib/gen'

	import { page } from '$app/stores'
	import FlowBuilder from '$lib/components/FlowBuilder.svelte'
	import { initialArgsStore, workspaceStore } from '$lib/stores'
	import {
		cleanValueProperties,
		decodeState,
		emptySchema,
		orderedJsonStringify,
		type StateStore
	} from '$lib/utils'
	import { initFlow } from '$lib/components/flows/flowStore.svelte'
	import { goto } from '$lib/navigation'
	import { afterNavigate, replaceState } from '$app/navigation'
	import { writable } from 'svelte/store'
	import type { FlowState } from '$lib/components/flows/flowState'
	import { sendUserToast } from '$lib/toast'
	import DiffDrawer from '$lib/components/DiffDrawer.svelte'
	import UnsavedConfirmationModal from '$lib/components/common/confirmationModal/UnsavedConfirmationModal.svelte'
	import type { ScheduleTrigger } from '$lib/components/triggers'
	import type { Trigger } from '$lib/components/triggers/utils'
	import { untrack } from 'svelte'

	let version: undefined | number = $state(undefined)
	let nodraft = $page.url.searchParams.get('nodraft')
	const initialState = nodraft ? undefined : localStorage.getItem(`flow-${$page.params.path}`)
	let stateLoadedFromUrl = initialState != undefined ? decodeState(initialState) : undefined
	let initialArgs = $state({})
	if ($initialArgsStore) {
		initialArgs = $initialArgsStore
		$initialArgsStore = undefined
	}

	let savedFlow:
		| (Flow & {
				draft?: Flow | undefined
		  })
		| undefined = $state(undefined)

	afterNavigate(() => {
		if (nodraft) {
			let url = new URL($page.url.href)
			url.search = ''
			replaceState(url.toString(), $page.state)
		}
	})

	export const flowStore: StateStore<Flow> = $state({
		val: {
			summary: '',
			value: { modules: [] },
			path: '',
			edited_at: '',
			edited_by: '',
			archived: false,
			extra_perms: {},
			schema: emptySchema()
		}
	})
	const flowStateStore = writable<FlowState>({})

	let loading = $state(false)

	let selectedId: string = $state('settings-metadata')

	let nobackenddraft = false

	let savedPrimarySchedule: ScheduleTrigger | undefined = $state(
		stateLoadedFromUrl?.primarySchedule
	)

	let draftTriggersFromUrl: Trigger[] | undefined = $state(undefined)
	let selectedTriggerIndexFromUrl: number | undefined = $state(undefined)

	let flowBuilder: FlowBuilder | undefined = $state(undefined)

	async function loadFlow(): Promise<void> {
		console.log('loadFlow')
		loading = true
		let flow: Flow
		let statePath = stateLoadedFromUrl?.path
		if (stateLoadedFromUrl != undefined && statePath == $page.params.path) {
			// Currently there is no way to get version of flow with flow.
			// So we have to request it here
			version = (
				await FlowService.getFlowLatestVersion({
					workspace: $workspaceStore!,
					path: statePath
				})
			).id

			savedFlow = await FlowService.getFlowByPathWithDraft({
				workspace: $workspaceStore!,
				path: statePath
			})

			const draftOrDeployed = cleanValueProperties(savedFlow?.draft || savedFlow)
			const urlScript = cleanValueProperties(
				$state.snapshot({
					...stateLoadedFromUrl.flow,
					draft_triggers: stateLoadedFromUrl.draft_triggers
				})
			)
			flow = stateLoadedFromUrl.flow
			draftTriggersFromUrl = stateLoadedFromUrl.draft_triggers
			selectedTriggerIndexFromUrl = stateLoadedFromUrl.selected_trigger
			flowBuilder?.setDraftTriggers(draftTriggersFromUrl)
			flowBuilder?.setSelectedTriggerIndex(selectedTriggerIndexFromUrl)
			const selectedId = stateLoadedFromUrl?.selectedId ?? 'settings-metadata'
			const reloadAction = () => {
				stateLoadedFromUrl = undefined
				goto(`/flows/edit/${statePath}?selected=${selectedId}`)
				loadFlow()
			}
			if (orderedJsonStringify(draftOrDeployed) === orderedJsonStringify(urlScript)) {
				reloadAction()
			} else {
				sendUserToast('Flow loaded from browser storage', false, [
					{
						label: 'Discard browser stored autosave and reload',
						callback: reloadAction
					},
					{
						label: 'Show diff',
						callback: async () => {
							diffDrawer?.openDrawer()
							diffDrawer?.setDiff({
								mode: 'simple',
								original: draftOrDeployed,
								current: urlScript,
								title: `${savedFlow?.draft ? 'Latest saved draft' : 'Deployed'} <> Autosave`,
								button: { text: 'Discard autosave', onClick: reloadAction }
							})
						}
					}
				])
			}
		} else {
			// Currently there is no way to get version of flow with flow.
			// So we have to request it here
			version = (
				await FlowService.getFlowLatestVersion({
					workspace: $workspaceStore!,
					path: $page.params.path
				})
			).id

			const flowWithDraft = await FlowService.getFlowByPathWithDraft({
				workspace: $workspaceStore!,
				path: $page.params.path
			})
			savedFlow = {
				...structuredClone($state.snapshot(flowWithDraft)),
				draft: flowWithDraft.draft
					? {
							...structuredClone($state.snapshot(flowWithDraft.draft)),
							path: flowWithDraft.draft.path ?? flowWithDraft.path // backward compatibility for old drafts missing path
						}
					: undefined
			} as Flow & {
				draft?: Flow & {
					draft_triggers?: Trigger[]
				}
			}
			if (flowWithDraft.draft != undefined && !nobackenddraft) {
				flow = flowWithDraft.draft
				savedPrimarySchedule = flowWithDraft?.draft?.['primary_schedule']
				flowBuilder?.setPrimarySchedule(savedPrimarySchedule)
				flowBuilder?.setDraftTriggers(flowWithDraft?.draft?.['draft_triggers'])

				if (!flowWithDraft.draft_only) {
					const deployed = cleanValueProperties(flowWithDraft)
					const draft = cleanValueProperties(flow)
					const reloadAction = async () => {
						stateLoadedFromUrl = undefined
						await DraftService.deleteDraft({
							workspace: $workspaceStore!,
							kind: 'flow',
							path: flow.path
						})
						nobackenddraft = true
						loadFlow()
					}
					sendUserToast('flow loaded from latest saved draft', false, [
						{
							label: 'Discard draft and load from latest deployed version',
							callback: reloadAction
						},
						{
							label: 'Show diff',
							callback: async () => {
								diffDrawer?.openDrawer()
								diffDrawer?.setDiff({
									mode: 'simple',
									original: deployed,
									current: draft,
									title: 'Deployed <> Draft',
									button: { text: 'Discard draft', onClick: reloadAction }
								})
							}
						}
					])
				}
			} else {
				flow = flowWithDraft
				flowBuilder?.setDraftTriggers(undefined)
			}
		}

		await initFlow(flow, flowStore, flowStateStore)
		loading = false
		selectedId = stateLoadedFromUrl?.selectedId ?? $page.url.searchParams.get('selected')
	}

	$effect(() => {
		if ($workspaceStore) {
			untrack(() => loadFlow())
		}
	})

	let diffDrawer: DiffDrawer | undefined = $state()

	async function restoreDraft() {
		if (!savedFlow || !savedFlow.draft) {
			sendUserToast('Could not restore to draft', true)
			return
		}
		diffDrawer?.closeDrawer()
		stateLoadedFromUrl = undefined
		goto(`/flows/edit/${savedFlow.draft.path}`)
		loadFlow()
	}

	async function restoreDeployed() {
		if (!savedFlow) {
			sendUserToast('Could not restore to deployed', true)
			return
		}
		diffDrawer?.closeDrawer()
		if (savedFlow.draft) {
			await DraftService.deleteDraft({
				workspace: $workspaceStore!,
				kind: 'flow',
				path: savedFlow.path
			})
		}
		stateLoadedFromUrl = undefined
		goto(`/flows/edit/${savedFlow.path}`)
		loadFlow()
	}
</script>

<!-- <div id="monaco-widgets-root" class="monaco-editor" style="z-index: 1200;" /> -->

<DiffDrawer bind:this={diffDrawer} {restoreDeployed} {restoreDraft} />
<FlowBuilder
	on:deploy={(e) => {
		goto(`/flows/get/${e.detail}?workspace=${$workspaceStore}`)
	}}
	on:details={(e) => {
		goto(`/flows/get/${e.detail}?workspace=${$workspaceStore}`)
	}}
	on:saveDraftOnlyAtNewPath={(e) => {
		const { path, selectedId } = e.detail
		goto(`/flows/edit/${path}?selected=${selectedId}`)
	}}
	on:historyRestore={() => {
		loadFlow()
	}}
	{flowStore}
	{flowStateStore}
	initialPath={$page.params.path}
	newFlow={false}
	{selectedId}
	{initialArgs}
	{loading}
	bind:this={flowBuilder}
	bind:savedFlow
	{diffDrawer}
	{savedPrimarySchedule}
	{draftTriggersFromUrl}
	{selectedTriggerIndexFromUrl}
	{version}
>
	<UnsavedConfirmationModal
		{diffDrawer}
		getInitialAndModifiedValues={flowBuilder?.getInitialAndModifiedValues}
	/>
</FlowBuilder>
