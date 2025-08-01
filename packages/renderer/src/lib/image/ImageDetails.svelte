<script lang="ts">
import type { ImageInfo } from '@podman-desktop/api';
import { StatusIcon, Tab } from '@podman-desktop/ui-svelte';
import { onDestroy, onMount } from 'svelte';
import type { Unsubscriber } from 'svelte/store';
import { router } from 'tinro';

import { containersInfos } from '/@/stores/containers';
import { context } from '/@/stores/context';
import { imageCheckerProviders } from '/@/stores/image-checker-providers';
import { imageFilesProviders } from '/@/stores/image-files-providers';
import { viewsContributions } from '/@/stores/views';
import type { ViewInfoUI } from '/@api/view-info';

import Route from '../../Route.svelte';
import { imagesInfos } from '../../stores/images';
import type { ContextUI } from '../context/context';
import Badge from '../ui/Badge.svelte';
import DetailsPage from '../ui/DetailsPage.svelte';
import { getTabUrl, isTabSelected } from '../ui/Util';
import {
  IMAGE_DETAILS_VIEW_BADGES,
  IMAGE_DETAILS_VIEW_ICONS,
  IMAGE_VIEW_BADGES,
  IMAGE_VIEW_ICONS,
} from '../view/views';
import { ImageUtils } from './image-utils';
import ImageActions from './ImageActions.svelte';
import ImageDetailsCheck from './ImageDetailsCheck.svelte';
import ImageDetailsFiles from './ImageDetailsFiles.svelte';
import ImageDetailsHistory from './ImageDetailsHistory.svelte';
import ImageDetailsInspect from './ImageDetailsInspect.svelte';
import ImageDetailsSummary from './ImageDetailsSummary.svelte';
import type { ImageInfoUI } from './ImageInfoUI';
import PushImageModal from './PushImageModal.svelte';
import RenameImageModal from './RenameImageModal.svelte';

interface Props {
  imageID: string;
  engineId: string;
  base64RepoTag: string;
}

let { imageID, engineId, base64RepoTag }: Props = $props();

let globalContext: ContextUI;
let viewContributions: ViewInfoUI[] = [];
let allImages: ImageInfo[];

const imageUtils = new ImageUtils();

let pushImageModal = $state(false);
function handlePushImageModal(): void {
  pushImageModal = true;
}

let renameImageModal = $state(false);
function handleRenameImageModal(): void {
  renameImageModal = true;
}

function closeModals(): void {
  pushImageModal = false;
  renameImageModal = false;
}

let imageInfo: ImageInfo | undefined = $state();
let image: ImageInfoUI | undefined = $state();
let detailsPage: DetailsPage | undefined = $state();

let showCheckTab: boolean = $state(false);
let showFilesTab: boolean = $state(false);
let checkerProvidersUnsubscribe: Unsubscriber;
let filesProvidersUnsubscribe: Unsubscriber;
let viewsUnsubscribe: Unsubscriber;
let contextsUnsubscribe: Unsubscriber;

function updateImage(): void {
  if (!allImages) {
    return;
  }
  imageInfo = allImages.find(c => c.Id === imageID && c.engineId === engineId);
  let tempImage;
  if (imageInfo) {
    tempImage = imageUtils.getImageInfoUI(imageInfo, base64RepoTag, $containersInfos, globalContext, viewContributions);
  }
  if (tempImage) {
    image = tempImage;
  } else {
    // the image has been deleted
    detailsPage?.close();
  }
}

onMount(() => {
  checkerProvidersUnsubscribe = imageCheckerProviders.subscribe(providers => {
    showCheckTab = providers.length > 0;
  });

  filesProvidersUnsubscribe = imageFilesProviders.subscribe(providers => {
    showFilesTab = providers.length > 0;
  });

  viewsUnsubscribe = viewsContributions.subscribe(value => {
    viewContributions =
      value.filter(
        view =>
          view.viewId === IMAGE_DETAILS_VIEW_ICONS ||
          view.viewId === IMAGE_VIEW_ICONS ||
          view.viewId === IMAGE_VIEW_BADGES ||
          view.viewId === IMAGE_DETAILS_VIEW_BADGES,
      ) || [];
    updateImage();
  });

  contextsUnsubscribe = context.subscribe(value => {
    globalContext = value;
    updateImage();
  });

  // loading image info
  return imagesInfos.subscribe(images => {
    allImages = images;
    updateImage();
  });
});

onDestroy(() => {
  // unsubscribe from the store
  checkerProvidersUnsubscribe?.();
  filesProvidersUnsubscribe?.();
  viewsUnsubscribe?.();
  contextsUnsubscribe?.();
});
</script>

{#if image}
  <DetailsPage title={image.name} titleDetail={image.shortId} subtitle={image.tag} bind:this={detailsPage}>
    {#snippet iconSnippet()}
      {#if image}<StatusIcon icon={image.icon} size={24} status={image.status} />{/if}
    {/snippet}
    {#snippet subtitleSnippet()}
      {#if image?.badges.length}
        <div class="flex flex-row">
          {#each image.badges as badge, index (index)}
            <Badge color={badge.color} label={badge.label} />
          {/each}
        </div>
      {/if}
    {/snippet}
    {#snippet actionsSnippet()}
      {#if image}
        <ImageActions
          image={image}
          onPushImage={handlePushImageModal}
          onRenameImage={handleRenameImageModal}
          detailed={true}
          dropdownMenu={false}
          groupContributions={true}
          on:update={(): ImageInfoUI | undefined => (image = image)} />
      {/if}
    {/snippet}
    {#snippet tabsSnippet()}
      <Tab title="Summary" selected={isTabSelected($router.path, 'summary')} url={getTabUrl($router.path, 'summary')} />
      <Tab title="History" selected={isTabSelected($router.path, 'history')} url={getTabUrl($router.path, 'history')} />
      <Tab title="Inspect" selected={isTabSelected($router.path, 'inspect')} url={getTabUrl($router.path, 'inspect')} />
      {#if showCheckTab}
        <Tab title="Check" selected={isTabSelected($router.path, 'check')} url={getTabUrl($router.path, 'check')} />
      {/if}
      {#if showFilesTab}
        <Tab title="Files" selected={isTabSelected($router.path, 'files')} url={getTabUrl($router.path, 'files')} />
      {/if}
    {/snippet}
    {#snippet contentSnippet()}
      {#if image}
        <Route path="/summary" breadcrumb="Summary" navigationHint="tab">
          <ImageDetailsSummary image={image} />
        </Route>
        <Route path="/history" breadcrumb="History" navigationHint="tab">
          <ImageDetailsHistory image={image} />
        </Route>
        <Route path="/inspect" breadcrumb="Inspect" navigationHint="tab">
          <ImageDetailsInspect image={image} />
        </Route>
        <Route path="/check" breadcrumb="Check" navigationHint="tab">
          <ImageDetailsCheck imageInfo={imageInfo} />
        </Route>
        <Route path="/files" breadcrumb="Files" navigationHint="tab">
          <ImageDetailsFiles imageInfo={imageInfo} />
        </Route>
      {/if}
    {/snippet}
  </DetailsPage>

  {#if pushImageModal}
    <PushImageModal
      imageInfoToPush={image}
      closeCallback={closeModals} />
  {/if}
  {#if renameImageModal}
    <RenameImageModal
      imageInfoToRename={image}
      detailed={true}
      closeCallback={closeModals} />
  {/if}
{/if}
