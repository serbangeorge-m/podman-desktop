<script lang="ts">
import { faStop } from '@fortawesome/free-solid-svg-icons';

import type { CombinedExtensionInfoUI } from '/@/stores/all-installed-extensions';

import LoadingIconButton from '../ui/LoadingIconButton.svelte';

interface Props {
  extension: CombinedExtensionInfoUI;
}

let { extension }: Props = $props();

let inProgress = $state(false);

async function stopExtension(): Promise<void> {
  inProgress = true;
  await window.stopExtension(extension.id);
  inProgress = false;
}
</script>

{#if extension.state === 'started' || extension.state === 'starting'}
  <LoadingIconButton
    clickAction={stopExtension}
    action="stop"
    icon={faStop}
    state={{ status: extension.type === 'dd' ? 'unsupported' : extension.state, inProgress }}
    leftPosition="left-[0.15rem]" />
{/if}
