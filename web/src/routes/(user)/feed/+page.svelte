<script lang="ts">
  import UserPageLayout from '$lib/components/layouts/UserPageLayout.svelte';
  import { getAssetMediaUrl, getAssetPlaybackUrl } from '$lib/utils';
  import { handleError } from '$lib/utils/handle-error';
  import { AssetMediaSize, AssetOrder, AssetTypeEnum, AssetVisibility, searchAssets, searchRandom } from '@immich/sdk';
  import { LoadingSpinner, Icon } from '@immich/ui';
  import { mdiRefresh, mdiSortCalendarDescending, mdiShuffle } from '@mdi/js';
  import { onDestroy, untrack } from 'svelte';
  import { t } from 'svelte-i18n';
  import type { PageData } from './$types';
  import type { AssetResponseDto } from '@immich/sdk';

  type Props = { data: PageData };
  let { data }: Props = $props();

  type SortMode = 'date' | 'random';

  let sortMode = $state<SortMode>('date');
  let assets = $state<AssetResponseDto[]>([]);
  let isLoading = $state(false);
  let cursor = $state<string | undefined>(undefined);
  let hasMore = $state(true);
  let sentinel = $state<HTMLDivElement | undefined>(undefined);
  let observer: IntersectionObserver | undefined;

  const PAGE_SIZE = 20;

  const loadAssets = async (reset = false) => {
    if (isLoading) {
      return;
    }
    if (!reset && !hasMore) {
      return;
    }

    isLoading = true;

    try {
      if (sortMode === 'random') {
        const results = await searchRandom({
          randomSearchDto: {
            size: PAGE_SIZE,
            visibility: AssetVisibility.Timeline,
            withExif: true,
          },
        });
        if (reset) {
          assets = results;
        } else {
          assets = [...assets, ...results];
        }
        hasMore = results.length === PAGE_SIZE;
        cursor = undefined;
      } else {
        const response = await searchAssets({
          metadataSearchDto: {
            size: PAGE_SIZE,
            order: AssetOrder.Desc,
            visibility: AssetVisibility.Timeline,
            cursor: reset ? undefined : cursor,
            withExif: true,
          },
        });
        const newAssets = response.assets.items;
        cursor = response.assets.nextPage ?? undefined;
        hasMore = !!response.assets.nextPage;
        if (reset) {
          assets = newAssets;
        } else {
          assets = [...assets, ...newAssets];
        }
      }
    } catch (error) {
      handleError(error, $t('errors.unable_to_load_assets'));
    } finally {
      isLoading = false;
    }
  };

  const setSortMode = async (mode: SortMode) => {
    if (sortMode === mode) {
      return;
    }
    sortMode = mode;
    cursor = undefined;
    hasMore = true;
    await loadAssets(true);
  };

  const handleRefresh = async () => {
    cursor = undefined;
    hasMore = true;
    await loadAssets(true);
  };

  $effect(() => {
    untrack(() => void loadAssets(true));
  });

  $effect(() => {
    if (!sentinel) {
      return;
    }
    observer?.disconnect();
    observer = new IntersectionObserver(
      (entries) => {
        if (entries[0]?.isIntersecting && !isLoading && hasMore) {
          void loadAssets(false);
        }
      },
      { rootMargin: '400px' },
    );
    observer.observe(sentinel);
    return () => observer?.disconnect();
  });

  onDestroy(() => observer?.disconnect());

  const getVideoUrl = (asset: AssetResponseDto) =>
    getAssetPlaybackUrl({ id: asset.id, cacheKey: asset.thumbhash });

  const getImageUrl = (asset: AssetResponseDto) =>
    getAssetMediaUrl({ id: asset.id, size: AssetMediaSize.Preview, cacheKey: asset.thumbhash });

  const formatDate = (dateStr: string) => {
    return new Date(dateStr).toLocaleDateString(undefined, {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
    });
  };
</script>

<UserPageLayout title={data.meta.title}>
  <div class="mx-auto flex w-full max-w-xl flex-col gap-2 px-2 py-4 pb-20">
    <div class="sticky top-0 z-10 flex items-center gap-2 rounded-xl bg-immich-bg/90 px-3 py-2 shadow backdrop-blur dark:bg-immich-dark-bg/90">
      <button
        class="flex items-center gap-1.5 rounded-lg px-3 py-1.5 text-sm font-medium transition-colors
          {sortMode === 'date'
          ? 'bg-immich-primary text-white dark:bg-immich-dark-primary'
          : 'text-immich-fg hover:bg-gray-100 dark:text-immich-dark-fg dark:hover:bg-gray-800'}"
        onclick={() => setSortMode('date')}
      >
        <Icon icon={mdiSortCalendarDescending} size="1em" />
        {$t('by_date')}
      </button>
      <button
        class="flex items-center gap-1.5 rounded-lg px-3 py-1.5 text-sm font-medium transition-colors
          {sortMode === 'random'
          ? 'bg-immich-primary text-white dark:bg-immich-dark-primary'
          : 'text-immich-fg hover:bg-gray-100 dark:text-immich-dark-fg dark:hover:bg-gray-800'}"
        onclick={() => setSortMode('random')}
      >
        <Icon icon={mdiShuffle} size="1em" />
        {$t('random')}
      </button>

      <span class="flex-1"></span>

      <button
        class="rounded-full p-1.5 text-immich-fg transition-colors hover:bg-gray-100 dark:text-immich-dark-fg dark:hover:bg-gray-800"
        title={$t('refresh')}
        onclick={handleRefresh}
      >
        <Icon icon={mdiRefresh} size="1.25em" />
      </button>
    </div>

    {#each assets as asset (asset.id)}
      <div class="overflow-hidden rounded-2xl bg-black shadow-md">
        <div class="relative w-full">
          {#if asset.type === AssetTypeEnum.Video}
            <video
              src={getVideoUrl(asset)}
              class="w-full"
              style="max-height: 80dvh; object-fit: contain;"
              autoplay
              muted
              loop
              playsinline
              controls
            ></video>
          {:else}
            <img
              src={getImageUrl(asset)}
              alt={asset.originalFileName}
              class="w-full"
              style="max-height: 80dvh; object-fit: contain;"
              loading="lazy"
            />
          {/if}
        </div>
        <div class="px-4 py-3 flex flex-col gap-1.5">
          <p class="text-xs text-gray-400">{formatDate(asset.localDateTime)}</p>
          {#if asset.exifInfo?.description}
            <p class="text-sm text-gray-200 leading-snug">{asset.exifInfo.description}</p>
          {/if}
          {#if asset.tags && asset.tags.length > 0}
            <div class="flex flex-wrap gap-1 pt-0.5">
              {#each asset.tags as tag (tag.id)}
                <span class="rounded-full bg-immich-primary/20 px-2 py-0.5 text-xs text-immich-primary dark:bg-immich-dark-primary/20 dark:text-immich-dark-primary">
                  {tag.value}
                </span>
              {/each}
            </div>
          {/if}
        </div>
      </div>
    {/each}

    <div bind:this={sentinel} class="flex justify-center py-6">
      {#if isLoading}
        <LoadingSpinner />
      {:else if !hasMore && assets.length > 0}
        <p class="text-sm text-gray-400">{$t('no_more_assets_to_show')}</p>
      {:else if !isLoading && assets.length === 0}
        <p class="text-sm text-gray-400">{$t('no_assets_message')}</p>
      {/if}
    </div>
  </div>
</UserPageLayout>
