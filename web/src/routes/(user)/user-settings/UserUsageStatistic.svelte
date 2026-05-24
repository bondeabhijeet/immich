<script lang="ts">
  import { locale } from '$lib/stores/preferences.store';
  import {
    AssetVisibility,
    getAlbumStatistics,
    getAssetStatistics,
    type AlbumStatisticsResponseDto,
    type AssetStatsResponseDto,
  } from '@immich/sdk';
  import { Heading, Table, TableBody, TableCell, TableHeader, TableHeading, TableRow } from '@immich/ui';
  import { onMount } from 'svelte';
  import { t } from 'svelte-i18n';

  let timelineStats: AssetStatsResponseDto = $state({
    videos: 0,
    images: 0,
    total: 0,
  });

  let favoriteStats: AssetStatsResponseDto = $state({
    videos: 0,
    images: 0,
    total: 0,
  });

  let archiveStats: AssetStatsResponseDto = $state({
    videos: 0,
    images: 0,
    total: 0,
  });

  let trashStats: AssetStatsResponseDto = $state({
    videos: 0,
    images: 0,
    total: 0,
  });

  let albumStats: AlbumStatisticsResponseDto = $state({
    owned: 0,
    shared: 0,
    notShared: 0,
  });
  let uploadHeatmap = $state<Array<{ date: string; count: number }>>([]);

  const getUploadHeatmap = async () => {
    const end = new Date();
    const start = new Date(end);
    start.setDate(end.getDate() - 364);

    const formatUtcDate = (date: Date) => date.toISOString().slice(0, 10);
    const from = formatUtcDate(start);
    const to = formatUtcDate(end);

    const response = await fetch(`/api/assets/statistics/uploads?from=${from}&to=${to}`);
    if (!response.ok) {
      return;
    }

    const data = (await response.json()) as { counts: Array<{ date: string; count: number }> };
    uploadHeatmap = data.counts ?? [];
  };

  const getUsage = async () => {
    [timelineStats, favoriteStats, archiveStats, trashStats, albumStats] = await Promise.all([
      getAssetStatistics({ visibility: AssetVisibility.Timeline }),
      getAssetStatistics({ isFavorite: true }),
      getAssetStatistics({ visibility: AssetVisibility.Archive }),
      getAssetStatistics({ isTrashed: true }),
      getAlbumStatistics(),
    ]);
  };

  onMount(async () => {
    await Promise.all([getUsage(), getUploadHeatmap()]);
  });

  const uploadHeatmapByDate = $derived(new Map(uploadHeatmap.map((item) => [item.date, item.count])));
  const maxDailyUploadCount = $derived(Math.max(...uploadHeatmap.map((item) => item.count), 0));

  const heatmapDates = $derived.by(() => {
    const today = new Date();
    const dates: string[] = [];
    for (let dayOffset = 364; dayOffset >= 0; dayOffset--) {
      const day = new Date(today);
      day.setDate(today.getDate() - dayOffset);
      dates.push(day.toISOString().slice(0, 10));
    }
    return dates;
  });

  const getHeatLevel = (count: number) => {
    if (count <= 0 || maxDailyUploadCount <= 0) {
      return 0;
    }
    const ratio = count / maxDailyUploadCount;
    if (ratio <= 0.25) return 1;
    if (ratio <= 0.5) return 2;
    if (ratio <= 0.75) return 3;
    return 4;
  };
</script>

{#snippet row(viewName: string, stats: AssetStatsResponseDto)}
  <TableRow>
    <TableCell class="w-1/4">{viewName}</TableCell>
    <TableCell class="w-1/4">{stats.images.toLocaleString($locale)}</TableCell>
    <TableCell class="w-1/4">{stats.videos.toLocaleString($locale)}</TableCell>
    <TableCell class="w-1/4">{stats.total.toLocaleString($locale)}</TableCell>
  </TableRow>
{/snippet}

<section class="my-4 w-full">
  <Heading size="tiny">{$t('photos_and_videos')}</Heading>
  <Table striped spacing="small" class="mt-4" size="small">
    <TableHeader>
      <TableHeading class="w-1/4">{$t('view_name')}</TableHeading>
      <TableHeading class="w-1/4">{$t('photos')}</TableHeading>
      <TableHeading class="w-1/4">{$t('videos')}</TableHeading>
      <TableHeading class="w-1/4">{$t('total')}</TableHeading>
    </TableHeader>
    <TableBody>
      {@render row($t('timeline'), timelineStats)}
      {@render row($t('favorites'), favoriteStats)}
      {@render row($t('archive'), archiveStats)}
      {@render row($t('trash'), trashStats)}
    </TableBody>
  </Table>

  <Heading size="tiny" class="mt-8">{$t('albums')}</Heading>
  <Table striped spacing="small" class="mt-4" size="small">
    <TableHeader>
      <TableHeading class="w-1/2">{$t('owned')}</TableHeading>
      <TableHeading class="w-1/2">{$t('shared')}</TableHeading>
    </TableHeader>
    <TableBody>
      <TableRow>
        <TableCell class="w-1/2">{albumStats.owned.toLocaleString($locale)}</TableCell>
        <TableCell class="w-1/2">{albumStats.shared.toLocaleString($locale)}</TableCell>
      </TableRow>
    </TableBody>
  </Table>
</section>

<section class="my-8 w-full">
  <Heading size="tiny">{$t('uploads')}</Heading>
  <div class="mt-4 grid grid-flow-col grid-rows-7 gap-1 overflow-x-auto pb-2">
    {#each heatmapDates as date}
      {@const count = uploadHeatmapByDate.get(date) ?? 0}
      {@const level = getHeatLevel(count)}
      <div
        class="h-3 w-3 rounded-sm border border-gray-200 dark:border-gray-700"
        class:bg-gray-100={level === 0}
        class:bg-green-200={level === 1}
        class:bg-green-400={level === 2}
        class:bg-green-600={level === 3}
        class:bg-green-800={level === 4}
        title={`${date}: ${count.toLocaleString($locale)} ${$t('uploads')}`}
      ></div>
    {/each}
  </div>
</section>
