<script lang="ts">
	import ReduxResult from '$components/ReduxResult.svelte';
	import ChangedFiles from '$components/v3/ChangedFiles.svelte';
	import CommitContextMenu, {
		type CommitMenuContext
	} from '$components/v3/CommitContextMenu.svelte';
	import CommitDetails from '$components/v3/CommitDetails.svelte';
	import CommitHeader from '$components/v3/CommitHeader.svelte';
	import CommitLine from '$components/v3/CommitLine.svelte';
	import CommitMessageEditor from '$components/v3/CommitMessageEditor.svelte';
	import ConflictResolutionConfirmModal from '$components/v3/ConflictResolutionConfirmModal.svelte';
	import Drawer from '$components/v3/Drawer.svelte';
	import KebabButton from '$components/v3/KebabButton.svelte';
	import { isLocalAndRemoteCommit } from '$components/v3/lib';
	import { writeClipboard } from '$lib/backend/clipboard';
	import { isCommit } from '$lib/branches/v3';
	import { CommitStatus, type CommitKey } from '$lib/commits/commit';
	import { DefaultForgeFactory } from '$lib/forge/forgeFactory.svelte';
	import { ModeService } from '$lib/mode/modeService';
	import { showToast } from '$lib/notifications/toasts';
	import { StackService } from '$lib/stacks/stackService.svelte';
	import { UiState } from '$lib/state/uiState.svelte';
	import { TestId } from '$lib/testing/testIds';
	import { splitMessage } from '$lib/utils/commitMessage';
	import { inject } from '@gitbutler/shared/context';
	import { getContext, maybeGetContext } from '@gitbutler/shared/context';
	import AsyncButton from '@gitbutler/ui/AsyncButton.svelte';
	import Button from '@gitbutler/ui/Button.svelte';
	import Icon from '@gitbutler/ui/Icon.svelte';
	import Tooltip from '@gitbutler/ui/Tooltip.svelte';

	type Props = {
		projectId: string;
		stackId: string;
		commitKey: CommitKey;
		active?: boolean;
		onerror: (err: unknown) => void;
	};

	const { projectId, stackId, commitKey, active, onerror }: Props = $props();

	const [stackService, uiState] = inject(StackService, UiState);

	let conflictResolutionConfirmationModal =
		$state<ReturnType<typeof ConflictResolutionConfirmModal>>();

	const forge = getContext(DefaultForgeFactory);
	const modeService = maybeGetContext(ModeService);
	const stackState = $derived(uiState.stack(stackId));
	const projectState = $derived(uiState.project(projectId));
	const commitTitle = $derived(projectState.commitTitle);
	const commitDescription = $derived(projectState.commitDescription);
	const selected = $derived(stackState.selection.get());
	const branchName = $derived(selected.current?.branchName);

	const commitResult = $derived(
		commitKey.upstream
			? stackService.upstreamCommitById(projectId, commitKey)
			: stackService.commitById(projectId, commitKey)
	);

	const changesResult = $derived(stackService.commitChanges(projectId, commitKey.commitId));

	const [updateCommitMessage, messageUpdateResult] = stackService.updateCommitMessage;

	type Mode = 'view' | 'edit';
	let commitMessageInput = $state<ReturnType<typeof CommitMessageEditor>>();

	function setMode(newMode: Mode) {
		switch (newMode) {
			case 'edit':
				projectState.editingCommitMessage.set(true);
				break;
			case 'view':
				projectState.editingCommitMessage.set(false);
				break;
		}
	}

	async function editCommitMessage() {
		if (!branchName) {
			throw new Error('No branch selected!');
		}
		if (!commitMessageInput) return;
		const commitMessage = commitMessageInput.getMessage();
		if (!commitMessage) {
			showToast({ message: 'Commit message is required', style: 'error' });
			return;
		}

		const newCommitId = await updateCommitMessage({
			projectId,
			stackId,
			commitId: commitKey.commitId,
			message: commitMessage
		});

		uiState.stack(stackId).selection.set({ branchName, commitId: newCommitId });
		setMode('view');
		commitTitle.current = '';
		commitDescription.current = '';
	}

	let commitMenuContext = $state<CommitMenuContext>();

	async function handleUncommit() {
		if (!branchName) return;
		await stackService.uncommit({ projectId, stackId, branchName, commitId: commitKey.commitId });
		projectState.drawerPage.set(undefined);
		if (branchName) stackState.selection.set({ branchName, commitId: undefined });
	}

	function canEdit() {
		return modeService !== undefined;
	}

	async function editPatch() {
		if (!canEdit()) return;
		await modeService!.enterEditMode(commitKey.commitId, stackId);
	}
</script>

<ReduxResult {stackId} {projectId} result={commitResult.current} {onerror}>
	{#snippet children(commit, env)}
		{@const isConflicted = isCommit(commit) && commit.hasConflicts}
		{#if projectState.editingCommitMessage.current}
			{@const parsedMessage = splitMessage(commit.message)}
			<Drawer
				testId={TestId.EditCommitMessageDrawer}
				projectId={env.projectId}
				stackId={env.stackId}
				title="Edit commit message"
				disableScroll
				minHeight={20}
			>
				<CommitMessageEditor
					bind:this={commitMessageInput}
					projectId={env.projectId}
					stackId={env.stackId}
					action={() => editCommitMessage()}
					actionLabel="Save"
					onCancel={() => setMode('view')}
					initialTitle={parsedMessage.title}
					initialMessage={parsedMessage.description}
					loading={messageUpdateResult.current.isLoading}
					existingCommitId={commit.id}
				/>
			</Drawer>
		{:else}
			<Drawer
				testId={TestId.CommitDrawer}
				projectId={env.projectId}
				stackId={env.stackId}
				noLeftPadding
			>
				{#snippet header()}
					<div class="commit-view__header text-13">
						{#if isLocalAndRemoteCommit(commit)}
							{@const commitState = commit.state}
							<CommitLine
								commitStatus={commitState.type}
								diverged={commitState.type === 'LocalAndRemote' &&
									commit.id !== commitState.subject}
								width={24}
							/>
						{:else}
							<CommitLine
								commitStatus="Remote"
								diverged={false}
								tooltip={CommitStatus.Remote}
								width={24}
							/>
						{/if}

						<div class="commit-view__header-title text-13">
							<Tooltip text="Copy commit SHA">
								<button
									type="button"
									class="commit-view__header-sha"
									onclick={() => {
										writeClipboard(commit.id, {
											message: 'Commit SHA copied'
										});
									}}
								>
									<span>
										{commit.id.substring(0, 7)}
									</span>
									<Icon name="copy-small" /></button
								>
							</Tooltip>
						</div>
					</div>
				{/snippet}

				{#snippet kebabMenu(header)}
					{@const data = isLocalAndRemoteCommit(commit)
						? {
								stackId,
								commitId: commit.id,
								commitMessage: commit.message,
								commitStatus: commit.state.type,
								commitUrl: forge.current.commitUrl(commit.id),
								onUncommitClick: () => handleUncommit(),
								onEditMessageClick: () => setMode('edit'),
								onPatchEditClick: () => editPatch()
							}
						: undefined}
					{#if data}
						<KebabButton
							contextElement={header}
							onclick={(element) => (commitMenuContext = { data, position: { element } })}
							oncontext={(coords) => (commitMenuContext = { data, position: { coords } })}
							activated={!!commitMenuContext?.position.element}
						/>
					{/if}
				{/snippet}

				<div class="commit-view">
					<CommitHeader
						commitMessage={commit.message}
						className="text-14 text-semibold text-body"
					/>
					<CommitDetails {commit}>
						<Button
							testId={TestId.CommitDrawerActionEditMessage}
							size="tag"
							kind="outline"
							icon="edit-small"
							onclick={() => {
								setMode('edit');
							}}
						>
							Edit message
						</Button>

						<AsyncButton
							testId={TestId.CommitDrawerActionUncommit}
							size="tag"
							kind="outline"
							icon="undo-small"
							action={async () => await handleUncommit()}
						>
							Uncommit
						</AsyncButton>

						{#if isConflicted}
							<AsyncButton
								size="tag"
								kind="solid"
								style="error"
								action={editPatch}
								icon="warning-small"
							>
								Resolve conflicts
							</AsyncButton>
						{:else}
							<AsyncButton size="tag" kind="outline" action={editPatch}>Edit commit</AsyncButton>
						{/if}
					</CommitDetails>
				</div>

				{#snippet filesSplitView()}
					<ReduxResult {projectId} {stackId} result={changesResult.current}>
						{#snippet children(changes)}
							<ChangedFiles
								title="Changed files"
								projectId={env.projectId}
								stackId={env.stackId}
								selectionId={{ type: 'commit', commitId: commit.id }}
								changes={changes.changes.filter(
									(change) => !(change.path in (changes.conflictEntries?.entries ?? {}))
								)}
								conflictEntries={changes.conflictEntries}
								{active}
							/>
						{/snippet}
					</ReduxResult>
				{/snippet}
			</Drawer>
		{/if}

		<ConflictResolutionConfirmModal
			bind:this={conflictResolutionConfirmationModal}
			onSubmit={editPatch}
		/>
	{/snippet}
</ReduxResult>

{#if commitMenuContext}
	<CommitContextMenu {projectId} bind:context={commitMenuContext} />
{/if}

<style>
	.commit-view {
		position: relative;
		height: 100%;
		flex: 1;
		display: flex;
		flex-direction: column;
		gap: 14px;
	}

	.commit-view__header {
		display: flex;
		gap: 8px;
		height: 100%;
		padding-left: 8px;
	}

	.commit-view__header-title {
		align-self: center;
	}

	.commit-view__header-sha {
		display: inline-flex;
		align-items: center;
		gap: 2px;
		text-decoration: dotted underline;
		transition: color var(--transition-fast);
		cursor: pointer;
		color: var(--clr-text-2);

		&:hover {
			color: var(--clr-text-1);
		}
	}
</style>
