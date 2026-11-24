<script lang="ts">
  import Icon from "#lib/components/Icon.svelte";
  import Button from "#lib/components/ui/Button.svelte";
  import IconButton from "#lib/components/ui/IconButton.svelte";
  import { languageName } from "#lib/shared/languages.ts";
  import type { SubtitleTrackSummary } from "#lib/shared/subtitles.ts";
  import type {
    TranslationOrigin,
    TranslationTask,
  } from "#lib/shared/translation.ts";
  import {
    isOriginal,
    newerTranslation,
    transcriptDate,
  } from "./transcripts.ts";

  type Props = {
    // A transcript that finished while another one was open.
    readyTrack: SubtitleTrackSummary | null;
    failedTranslations: TranslationTask[];
    openTrackId: string | null;
    tracks: SubtitleTrackSummary[];
    // Where the open transcript came from, when it is a translation.
    origin: TranslationOrigin | null;
    onopentrack: (trackId: string) => void;
    ondismissready: () => void;
    // Null when no original transcript is left to translate from.
    ontranslate: ((sourceId: string, languages: string[]) => void) | null;
  };

  let {
    readyTrack,
    failedTranslations,
    openTrackId,
    tracks,
    origin,
    onopentrack,
    ondismissready,
    ontranslate,
  }: Props = $props();

  const ROW_CLASSES =
    "flex items-center gap-2 border-b border-border py-1.5 pr-2 pl-4 text-sm text-text";

  const source = $derived(
    origin && tracks.find((track) => track.id === origin.sourceTrackId),
  );
  // The source's revision counts its saves, so a newer one means it was edited.
  const sourceChanged = $derived(
    Boolean(origin && source && source.revision > origin.sourceRevision),
  );
  // Two originals in one language are told apart by when they were made.
  const sourceName = $derived.by(() => {
    if (!source) return "";
    const name = languageName(source.language);
    const shared = tracks.some(
      (track) =>
        isOriginal(track) &&
        track.id !== source.id &&
        track.language === source.language,
    );
    return shared
      ? `${name} (${transcriptDate.format(source.createdAt)})`
      : name;
  });
  const openTrack = $derived(tracks.find((track) => track.id === openTrackId));
  const newer = $derived(
    openTrackId ? newerTranslation(tracks, openTrackId) : null,
  );
</script>

<!-- The live region stays mounted so screen readers announce a finished or failed job. -->
<div role="status" class="shrink-0">
  {#if readyTrack}
    <div class={ROW_CLASSES}>
      <span class="shrink-0 text-success" aria-hidden="true">
        <Icon name="check-circle" size={16} />
      </span>
      <p class="min-w-0 flex-1">
        The {languageName(readyTrack.language)} transcript is ready.
      </p>
      <Button size="sm" onclick={() => onopentrack(readyTrack.id)}>Open</Button>
      <IconButton
        label="Dismiss"
        icon="close"
        size="sm"
        onclick={ondismissready}
      />
    </div>
  {/if}
  {#each failedTranslations as task (task.jobId)}
    <div class={ROW_CLASSES}>
      <span class="shrink-0 text-danger" aria-hidden="true">
        <Icon name="alert" size={16} />
      </span>
      <p class="min-w-0 flex-1">
        The {languageName(task.targetLanguage)} translation failed{task.error
          ? `: ${task.error}`
          : "."}
      </p>
      {#if ontranslate}
        <Button
          size="sm"
          onclick={() =>
            ontranslate?.(task.sourceTrackId, [task.targetLanguage])}
        >
          Try again…
        </Button>
      {/if}
      <form method="POST" action="?/dismissTranslation">
        <input type="hidden" name="jobId" value={task.jobId} />
        <IconButton label="Dismiss" icon="close" size="sm" type="submit" />
      </form>
    </div>
  {/each}
</div>

{#if origin}
  <div
    class="flex shrink-0 items-center gap-2 border-b border-border bg-surface-sunken py-1.5 pr-2 pl-4 text-xs text-text-muted"
  >
    <span class="shrink-0" aria-hidden="true">
      <Icon name="languages" size={14} />
    </span>
    <p class="min-w-0 flex-1">
      Translated from
      {#if source}
        <button
          type="button"
          class="font-medium text-accent underline-offset-2 hover:underline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-focus"
          onclick={() => onopentrack(source.id)}
        >
          {sourceName}
        </button>
      {:else}
        {languageName(origin.sourceLanguage)}
      {/if}
      with {origin.modelLabel}.
      {#if newer}
        There's a newer {languageName(newer.language)} translation.
      {:else if sourceChanged}
        The {languageName(origin.sourceLanguage)} transcript changed since.
      {:else if origin.local}
        Read it through before publishing.
      {/if}
    </p>
    {#if newer}
      <Button size="sm" onclick={() => onopentrack(newer.id)}>Open</Button>
    {:else if sourceChanged && source && openTrack && ontranslate}
      <Button
        size="sm"
        onclick={() => ontranslate?.(source.id, [openTrack.language])}
      >
        Translate again…
      </Button>
    {/if}
  </div>
{/if}
