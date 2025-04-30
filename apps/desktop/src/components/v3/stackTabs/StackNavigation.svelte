<script lang="ts">
	import ButtonCreateNew from '$components/v3/stackTabs/ButtonCreateNew.svelte';
	import StackListItem from '$components/v3/stackTabs/StackListItem.svelte';
	import StackTab from '$components/v3/stackTabs/StackTab.svelte';
	import StackTabDraft from '$components/v3/stackTabs/StackTabDraft.svelte';
	import StackTabMenu from '$components/v3/stackTabs/StackTabMenu.svelte';
	import StackTabNew from '$components/v3/stackTabs/StackTabNew.svelte';
	import { stackPath } from '$lib/routes/routes.svelte';
	import { getStackBranchNames, getStackName, type Stack } from '$lib/stacks/stack';
	import { UiState } from '$lib/state/uiState.svelte';
	import { inject } from '@gitbutler/shared/context';
	import Button from '@gitbutler/ui/Button.svelte';
	import Icon from '@gitbutler/ui/Icon.svelte';
	import { onMount } from 'svelte';

	type Props = {
		projectId: string;
		stacks: Stack[];
		selectedId: string;
		isCommitting: boolean;
		width: number | undefined;
	};
	let { projectId, stacks, selectedId, isCommitting, width = $bindable() }: Props = $props();

	const [uiState] = inject(UiState);

	const stackState = $derived(uiState.stack(selectedId));
	const selection = $derived(stackState.selection.get());
	const selectedBranchName = $derived(selection.current?.branchName);

	let plusBtnEl = $state<HTMLButtonElement>();
	let tabsEl = $state<HTMLDivElement>();
	let scrollerEl = $state<HTMLDivElement>();

	// let scrollable = $state(false);
	// let scrolled = $state(false);
	// let scrolledEnd = $state(false);

	let panelFolded = $state(false);

	// function onscroll() {
	// 	scrolled = scrollerEl && scrollerEl.scrollLeft > 0 ? true : false;
	// 	scrolledEnd = scrollerEl
	// 		? scrollerEl.scrollLeft + scrollerEl.offsetWidth >= scrollerEl.scrollWidth
	// 		: false;
	// }

	// onMount(() => {
	// 	const observer = new ResizeObserver(() => {
	// 		scrollable = scrollerEl ? scrollerEl.scrollWidth > scrollerEl.offsetWidth : false;
	// 		width = tabsEl?.offsetWidth;
	// 	});
	// 	observer.observe(tabsEl!);
	// 	return () => {
	// 		observer.disconnect();
	// 	};
	// });
</script>

<div class="stacks-nav" bind:this={tabsEl}>
	<div class="stacks-nav__header" class:unfolded={!panelFolded}>
		<button
			type="button"
			class="stacks-nav__header-title"
			onclick={() => (panelFolded = !panelFolded)}
		>
			<div class="stacks-nav__header-chevron" class:folded={panelFolded}>
				<Icon name="chevron-right" />
			</div>
			<h4 class="text-14 text-semibold">Applied branches</h4>
		</button>

		<ButtonCreateNew
			bind:el={plusBtnEl}
			{scrollerEl}
			{projectId}
			stackId={selectedId}
			noStacks={stacks.length === 0}
		/>
	</div>

	{#if !panelFolded}
		{#if stacks.length > 0}
			{#each stacks as stack (getStackName(stack))}
				{@const branchNames = getStackBranchNames(stack)}

				{@const itemProps = {
					projectId,
					stackId: stack.id,
					href: stackPath(projectId, stack.id)
				}}

				{#if branchNames.length === 1}
					<StackListItem
						selected={branchNames[0] === selectedBranchName}
						{...itemProps}
						type="branch"
						name={branchNames[0]}
					></StackListItem>
				{:else if branchNames.length > 1}
					{@const restBranchNames = branchNames.slice(1)}
					<div class="stacks-nav__stack-group">
						<StackListItem
							selected={branchNames[0] === selectedBranchName}
							{...itemProps}
							type="stack-head"
							name={getStackName(stack)}
						/>
						{#each restBranchNames as branchName, j}
							<StackListItem
								{...itemProps}
								selected={branchName === selectedBranchName}
								type="stack-branch"
								name={branchName}
								isLast={j === restBranchNames.length - 1}
							/>
						{/each}
					</div>
				{/if}
			{/each}
		{/if}
	{/if}

	<!-- {#if stacks.length > 0}
		<div class="inner">
			<div class="scroller" bind:this={scrollerEl} class:scrolled {onscroll}>
				{#each stacks as stack, i (getStackName(stack))}
					{@const stackName = getStackName(stack)}
					{@const branchNames = getStackBranchNames(stack)}

					{@const first = i === 0}
					{@const last = i === stacks.length - 1}
					{@const selected = stack.id === selectedId}

					<StackTab
						name={stackName}
						{projectId}
						stackId={stack.id}
						href={stackPath(projectId, stack.id)}
						anchors={branchNames.slice(1)}
						{selected}
						onNextTab={() => {
							if (last) {
								plusBtnEl?.focus();
							}
						}}
						onPrevTab={() => {
							if (first) {
								plusBtnEl?.focus();
							}
						}}
					/>
				{/each}
			</div>
			<div class="shadow shadow-left" class:scrolled></div>
			<div class="shadow shadow-right" class:scrollable class:scrolled-end={scrolledEnd}></div>
		</div>
	{/if} -->

	<!-- {#if isCommitting && stacks.length === 0}
		<StackTabDraft />
	{:else}
		<StackTabNew
			bind:el={plusBtnEl}
			{scrollerEl}
			{projectId}
			stackId={selectedId}
			noStacks={stacks.length === 0}
		/>
	{/if} -->
</div>

<style lang="postcss">
	.stacks-nav {
		display: flex;
		flex-direction: column;
		width: 100%;
		border: 1px solid var(--clr-border-2);
		border-bottom: none;
		border-radius: var(--radius-ml) var(--radius-ml) 0 0;
		overflow: hidden;
	}

	.stacks-nav__header {
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 8px 8px 8px 14px;
		height: 44px;
		background-color: var(--clr-bg-1);

		&.unfolded {
			border-bottom: 1px solid var(--clr-border-2);
		}
	}

	.stacks-nav__header-title {
		display: flex;
		align-items: center;

		&:hover,
		&:focus {
			& > .stacks-nav__header-chevron {
				--chevron-color: var(--clr-text-1);
			}
		}
	}

	.stacks-nav__header-chevron {
		--chevron-rotate: 90deg;
		--chevron-color: var(--clr-text-2);
		display: flex;
		align-items: center;
		justify-content: center;
		color: var(--chevron-color);
		margin-right: 10px;
		opacity: 0.8;
		transform: rotate(var(--chevron-rotate));
		transition:
			color var(--transition-fast),
			transform var(--transition-medium);
	}

	.stacks-nav__header-chevron.folded {
		--chevron-rotate: 0deg;
	}

	.stacks-nav__list {
		display: flex;
		flex-direction: column;
		flex: 1;
		overflow: hidden;
		background-color: var(--clr-bg-1);
		border-top: 1px solid var(--clr-border-3);
	}

	.stacks-nav__stack-group {
		display: flex;
		flex-direction: column;
		border-bottom: 1px solid var(--clr-border-2);
	}
</style>
