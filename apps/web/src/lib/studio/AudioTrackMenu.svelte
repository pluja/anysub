<script lang="ts">
  import Icon from "#lib/components/Icon.svelte";
  import Menu from "#lib/components/ui/Menu.svelte";
  import MenuItem from "#lib/components/ui/MenuItem.svelte";
  import {
    audioTrackDetails,
    audioTrackName,
    type AudioTrack,
  } from "#lib/shared/audio-tracks.ts";
  import type { AudioTrackState } from "./project-view";

  type Props = {
    audioTracks: AudioTrack[];
    // The track the studio means to play, which may still be preparing.
    wanted: number;
    // The open transcript's track, or null without a transcript.
    transcriptTrack: number | null;
    stateOf: (audioTrack: number) => AudioTrackState;
    onchoose: (audioTrack: number) => void;
  };

  let { audioTracks, wanted, transcriptTrack, stateOf, onchoose }: Props =
    $props();

  let wantedTrack = $derived(
    audioTracks.find((track) => track.index === wanted) ?? audioTracks[0],
  );
  let wantedState = $derived(stateOf(wanted));
  let isOtherThanTranscript = $derived(
    transcriptTrack !== null && wanted !== transcriptTrack,
  );

  function stateLabel(state: AudioTrackState): string | null {
    switch (state.kind) {
      case "preparing":
        return state.progress === null
          ? "Preparing"
          : `Preparing ${Math.round(state.progress * 100)}%`;
      case "failed":
        return "Failed";
      case "missing":
        return "Not ready";
      case "ready":
        return null;
    }
  }
</script>

<Menu label="Audio" align="end">
  {#snippet trigger(args)}
    <button
      type="button"
      {...args}
      title={isOtherThanTranscript
        ? "Not the audio the open transcript came from"
        : undefined}
      class="flex min-w-0 items-center gap-1.5 rounded-sm px-2 py-1.5 text-xs font-medium hover:bg-surface-sunken focus-visible:outline-2 focus-visible:outline-focus {isOtherThanTranscript
        ? 'text-accent'
        : 'text-text'}"
    >
      <span class="shrink-0" aria-hidden="true">
        <Icon name="audio-lines" size={16} />
      </span>
      <span class="sr-only">Audio:</span>
      <span class="truncate @max-xl:sr-only">
        {audioTrackName(wantedTrack)}
      </span>
      {#if wantedState.kind !== "ready"}
        <span class="shrink-0 text-text-muted @max-xl:sr-only">
          · {stateLabel(wantedState)}
        </span>
      {/if}
      {#if isOtherThanTranscript}
        <span class="sr-only">, not the transcript's audio</span>
      {/if}
      <span class="shrink-0 text-text-muted" aria-hidden="true">
        <Icon name="chevron-down" size={14} />
      </span>
    </button>
  {/snippet}
  {#each audioTracks as track (track.index)}
    {@const label = stateLabel(stateOf(track.index))}
    {@const details = audioTrackDetails(track)}
    <MenuItem
      role="menuitemradio"
      checked={track.index === wanted}
      onselect={() => onchoose(track.index)}
    >
      <span
        class="grid max-w-80 min-w-0 flex-1 grid-cols-[minmax(0,1fr)_auto] items-baseline gap-x-3"
      >
        <span class="truncate font-medium">{audioTrackName(track)}</span>
        <span class="text-xs text-text-muted">
          {#if track.index === transcriptTrack}Transcript{/if}
        </span>
        <span class="truncate text-xs text-text-muted">
          {[details, label].filter(Boolean).join(" · ")}
        </span>
      </span>
    </MenuItem>
  {/each}
</Menu>
