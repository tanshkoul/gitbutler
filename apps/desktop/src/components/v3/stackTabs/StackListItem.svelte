<script lang="ts">
	import StackTabMenu from '$components/v3/stackTabs/StackTabMenu.svelte';
	import Icon from '@gitbutler/ui/Icon.svelte';
	import { goto } from '$app/navigation';
	// import type { Snippet } from 'svelte';

	interface Props {
		type: 'stack-head' | 'stack-branch' | 'branch';
		projectId: string;
		stackId: string;
		name: string | undefined;
		href?: string;
		selected?: boolean;
		isLast?: boolean;
	}

	const { type, projectId, stackId, href, selected, name, isLast }: Props = $props();

	function getIconName() {
		switch (type) {
			case 'stack-head':
				return 'chain-link';
			case 'stack-branch':
				return 'chain-link';
			case 'branch':
				return 'branch-small';
			default:
				return 'branch-small';
		}
	}
</script>

<div
	class="stack-list-item"
	class:selected
	role="presentation"
	onclick={(e) => {
		e.preventDefault();
		e.stopPropagation();
		if (href) {
			goto(href, { replaceState: true, keepFocus: true });
		}
	}}
>
	<div class="stack-list-item__content">
		<div class="stack-list-item__icon">
			<Icon name={getIconName()} />
			{#if type === 'stack-branch' || type === 'branch'}
				<svg
					class="stack-icon-frame"
					class:branch={type === 'branch'}
					class:stack-branch={type === 'stack-branch'}
					width="18"
					height="18"
					viewBox="0 0 18 18"
					xmlns="http://www.w3.org/2000/svg"
				>
					<path
						d="M5 0.5H13C15.4853 0.5 17.5 2.51472 17.5 5V13C17.5 15.4853 15.4853 17.5 13 17.5H5C2.51472 17.5 0.5 15.4853 0.5 13V5C0.500001 2.51472 2.51472 0.5 5 0.5Z"
					/>
				</svg>
			{/if}

			{#if type === 'stack-head'}
				<svg
					width="18"
					height="21"
					class="stack-head-frame"
					viewBox="0 0 18 21"
					xmlns="http://www.w3.org/2000/svg"
				>
					<path
						d="M5.93945 2.52148C7.66558 0.920125 10.3344 0.920125 12.0605 2.52148L16.0605 6.23242C16.9783 7.08386 17.4999 8.27941 17.5 9.53125V16C17.5 18.4853 15.4853 20.5 13 20.5H5C2.51472 20.5 0.5 18.4853 0.5 16V9.53125C0.500102 8.27941 1.02174 7.08386 1.93945 6.23242L5.93945 2.52148Z"
					/>
				</svg>
			{/if}

			{#if type === 'stack-head' || (type === 'stack-branch' && !isLast)}
				<div class="stack-list-item__connector"></div>
			{/if}
		</div>
		<span class="text-13 text-semibold truncate">{name}</span>
	</div>

	{#if type === 'stack-head' || type === 'branch'}
		<div class="stack-list-item__menu">
			<StackTabMenu {projectId} {stackId} />
		</div>
	{/if}
</div>

<style lang="postcss">
	.stack-list-item {
		display: flex;
		align-items: center;
		height: 36px;
		padding: 0 10px 0 12px;
		background: var(--clr-bg-1);
		transition: var(--transition-fast);

		&:hover {
			background: var(--clr-bg-1-muted);
		}

		&.selected {
			background: var(--clr-bg-1-muted);
		}
	}

	.stack-list-item__content {
		flex: 1;
		display: flex;
		align-items: center;
	}

	.stack-list-item__icon {
		position: relative;
		display: flex;
		align-items: center;
		justify-content: center;
		width: 18px;
		height: 18px;
		color: var(--clr-text-2);
		margin-right: 10px;
	}

	.stack-icon-frame {
		z-index: 0;
		position: absolute;
		top: 0;
		left: 0;
		fill: var(--clr-bg-1);

		&.branch {
			stroke: var(--clr-border-1);
		}

		&.stack-branch {
			stroke: var(--clr-border-2);
		}
	}

	.stack-head-frame {
		z-index: 0;
		position: absolute;
		bottom: 0;
		left: 0;
		fill: var(--clr-bg-1);
		stroke: var(--clr-border-1);
	}

	.stack-list-item__connector {
		--connector-height: 18px;
		position: absolute;
		bottom: calc(var(--connector-height) * -1);
		left: 50%;
		width: 1px;
		height: var(--connector-height);
		background-color: var(--clr-border-1);
		transform: translateX(-50%);
	}

	.stack-list-item__menu {
		display: flex;
	}
</style>
