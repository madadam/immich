<script lang="ts">
  import {
    assetInteractionStore,
    assetSelectionCandidates,
    assetsInAlbumStoreState,
    isMultiSelectStoreState,
    selectedAssets,
    selectedGroup,
  } from '$lib/stores/asset-interaction.store';
  import { assetStore } from '$lib/stores/assets.store';
  import { locale } from '$lib/stores/preferences.store';
  import { getAssetRatio } from '$lib/utils/asset-utils';
  import { formatGroupTitle, splitBucketIntoDateGroups } from '$lib/utils/timeline-util';
  import type { AssetResponseDto } from '@api';
  import justifiedLayout from 'justified-layout';
  import { DateTime } from 'luxon';
  import { createEventDispatcher } from 'svelte';
  import CheckCircle from 'svelte-material-icons/CheckCircle.svelte';
  import CircleOutline from 'svelte-material-icons/CircleOutline.svelte';
  import { fly } from 'svelte/transition';
  import Thumbnail from '../assets/thumbnail/thumbnail.svelte';

  export let assets: AssetResponseDto[];
  export let bucketDate: string;
  export let bucketHeight: number;
  export let isAlbumSelectionMode = false;
  export let viewportWidth: number;

  const dispatch = createEventDispatcher();

  let isMouseOverGroup = false;
  let actualBucketHeight: number;
  let hoveredDateGroup = '';

  interface LayoutBox {
    top: number;
    left: number;
    width: number;
  }

  $: assetsGroupByDate = splitBucketIntoDateGroups(assets, $locale);

  $: geometry = (() => {
    const geometry = [];
    for (let group of assetsGroupByDate) {
      const justifiedLayoutResult = justifiedLayout(group.map(getAssetRatio), {
        boxSpacing: 2,
        containerWidth: Math.floor(viewportWidth),
        containerPadding: 0,
        targetRowHeightTolerance: 0.15,
        targetRowHeight: 235,
      });
      geometry.push({
        ...justifiedLayoutResult,
        containerWidth: calculateWidth(justifiedLayoutResult.boxes),
      });
    }
    return geometry;
  })();

  $: {
    if (actualBucketHeight && actualBucketHeight != 0 && actualBucketHeight != bucketHeight) {
      const heightDelta = assetStore.updateBucketHeight(bucketDate, actualBucketHeight);
      if (heightDelta !== 0) {
        scrollTimeline(heightDelta);
      }
    }
  }

  function scrollTimeline(heightDelta: number) {
    dispatch('shift', {
      heightDelta,
    });
  }

  const calculateWidth = (boxes: LayoutBox[]): number => {
    let width = 0;
    for (const box of boxes) {
      if (box.top < 100) {
        width = box.left + box.width;
      }
    }

    return width;
  };

  const assetClickHandler = (
    asset: AssetResponseDto,
    assetsInDateGroup: AssetResponseDto[],
    dateGroupTitle: string,
  ) => {
    if (isAlbumSelectionMode) {
      assetSelectHandler(asset, assetsInDateGroup, dateGroupTitle);
      return;
    }

    if ($isMultiSelectStoreState) {
      assetSelectHandler(asset, assetsInDateGroup, dateGroupTitle);
    } else {
      assetInteractionStore.setViewingAsset(asset);
    }
  };

  const selectAssetGroupHandler = (selectAssetGroupHandler: AssetResponseDto[], dateGroupTitle: string) => {
    if ($selectedGroup.has(dateGroupTitle)) {
      assetInteractionStore.removeGroupFromMultiselectGroup(dateGroupTitle);
      selectAssetGroupHandler.forEach((asset) => {
        assetInteractionStore.removeAssetFromMultiselectGroup(asset);
      });
    } else {
      assetInteractionStore.addGroupToMultiselectGroup(dateGroupTitle);
      selectAssetGroupHandler.forEach((asset) => {
        assetInteractionStore.addAssetToMultiselectGroup(asset);
      });
    }
  };

  const assetSelectHandler = (
    asset: AssetResponseDto,
    assetsInDateGroup: AssetResponseDto[],
    dateGroupTitle: string,
  ) => {
    dispatch('selectAssets', { asset });

    // Check if all assets are selected in a group to toggle the group selection's icon
    let selectedAssetsInGroupCount = assetsInDateGroup.filter((asset) => $selectedAssets.has(asset)).length;

    // if all assets are selected in a group, add the group to selected group
    if (selectedAssetsInGroupCount == assetsInDateGroup.length) {
      assetInteractionStore.addGroupToMultiselectGroup(dateGroupTitle);
    } else {
      assetInteractionStore.removeGroupFromMultiselectGroup(dateGroupTitle);
    }
  };

  const assetMouseEventHandler = (dateGroupTitle: string, asset: AssetResponseDto | null) => {
    // Show multi select icon on hover on date group
    hoveredDateGroup = dateGroupTitle;

    if ($isMultiSelectStoreState) {
      dispatch('selectAssetCandidates', { asset });
    }
  };
</script>

<section
  id="asset-group-by-date"
  class="flex flex-wrap gap-x-12"
  bind:clientHeight={actualBucketHeight}
  bind:clientWidth={viewportWidth}
>
  {#each assetsGroupByDate as assetsInDateGroup, groupIndex (assetsInDateGroup[0].id)}
    {@const dateGroupTitle = formatGroupTitle(DateTime.fromISO(assetsInDateGroup[0].fileCreatedAt).startOf('day'))}
    <!-- Asset Group By Date -->

    <div
      class="flex flex-col mt-5"
      on:mouseenter={() => {
        isMouseOverGroup = true;
        assetMouseEventHandler(dateGroupTitle, null);
      }}
      on:mouseleave={() => {
        isMouseOverGroup = false;
        assetMouseEventHandler(dateGroupTitle, null);
      }}
    >
      <!-- Date group title -->
      <p
        class="font-medium text-xs md:text-sm text-immich-fg dark:text-immich-dark-fg mb-2 flex place-items-center h-6"
        style="width: {geometry[groupIndex].containerWidth}px"
      >
        {#if (hoveredDateGroup == dateGroupTitle && isMouseOverGroup) || $selectedGroup.has(dateGroupTitle)}
          <div
            transition:fly={{ x: -24, duration: 200, opacity: 0.5 }}
            class="inline-block px-2 hover:cursor-pointer"
            on:click={() => selectAssetGroupHandler(assetsInDateGroup, dateGroupTitle)}
            on:keydown={() => selectAssetGroupHandler(assetsInDateGroup, dateGroupTitle)}
          >
            {#if $selectedGroup.has(dateGroupTitle)}
              <CheckCircle size="24" color="#4250af" />
            {:else}
              <CircleOutline size="24" color="#757575" />
            {/if}
          </div>
        {/if}

        <span class="first-letter:capitalize truncate" title={dateGroupTitle}>
          {dateGroupTitle}
        </span>
      </p>

      <!-- Image grid -->
      <div
        class="relative"
        style="height: {geometry[groupIndex].containerHeight}px;width: {geometry[groupIndex].containerWidth}px"
      >
        {#each assetsInDateGroup as asset, index (asset.id)}
          {@const box = geometry[groupIndex].boxes[index]}
          <div
            class="absolute"
            style="width: {box.width}px; height: {box.height}px; top: {box.top}px; left: {box.left}px"
          >
            <Thumbnail
              {asset}
              {groupIndex}
              on:click={() => assetClickHandler(asset, assetsInDateGroup, dateGroupTitle)}
              on:select={() => assetSelectHandler(asset, assetsInDateGroup, dateGroupTitle)}
              on:mouse-event={() => assetMouseEventHandler(dateGroupTitle, asset)}
              selected={$selectedAssets.has(asset) || $assetsInAlbumStoreState.some(({ id }) => id === asset.id)}
              selectionCandidate={$assetSelectionCandidates.has(asset)}
              disabled={$assetsInAlbumStoreState.some(({ id }) => id === asset.id)}
              thumbnailWidth={box.width}
              thumbnailHeight={box.height}
            />
          </div>
        {/each}
      </div>
    </div>
  {/each}
</section>

<style>
  #asset-group-by-date {
    contain: layout;
  }
</style>
