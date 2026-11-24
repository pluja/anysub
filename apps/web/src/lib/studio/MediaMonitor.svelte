<script lang="ts">
  import { untrack, type Snippet } from "svelte";
  import Icon from "#lib/components/Icon.svelte";
  import type { CueEditor } from "#lib/editor/editor.svelte.ts";
  import { projectMediaUrl } from "#lib/shared/routes.ts";
  import { isAudioMedia } from "#lib/shared/media.ts";
  import type { StudioData } from "#lib/shared/studio-data.ts";
  import PlayerControls from "./PlayerControls.svelte";
  import { derivativeUrl } from "./project-view";

  type Props = {
    project: StudioData["project"];
    derivatives: StudioData["derivatives"];
    // The audio track whose waveform and preview play.
    audioTrack: number;
    editor: CueEditor;
    audioMenu?: Snippet;
  };

  let { project, derivatives, audioTrack, editor, audioMenu }: Props = $props();

  let transport = $derived(editor.transport);
  let thumbnailUrl = $derived(derivativeUrl(derivatives, "thumbnail"));
  // A browser plays a file's first audio track, so the source stands in for
  // track 1 when it needs no preview.
  let previewUrl = $derived(
    derivativeUrl(derivatives, "proxy", audioTrack) ??
      projectMediaUrl(project.id),
  );
  let isVideo = $derived(
    project.hasSourceMedia &&
      !isAudioMedia(project.mediaType, project.mediaFilename),
  );
  let videoAspectRatio = $state(16 / 9);
  let volume = $state(1);
  let muted = $state(false);
  let monitor = $state<HTMLElement>();
  let isFullscreen = $state(false);

  // Switching tracks loads another file, which starts at the beginning, so
  // the playhead and playback carry over once it loads.
  let resumeFrom: {
    projectId: string;
    seconds: number;
    playing: boolean;
  } | null = null;
  $effect.pre(() => {
    void previewUrl;
    untrack(() => {
      const media = transport.mediaElement;
      if (!media || media.readyState === 0) return;
      resumeFrom = {
        projectId: project.id,
        seconds: media.currentTime,
        playing: !media.paused,
      };
    });
  });

  // Fullscreen is missing on iPhone.
  let canFullscreen = $state(false);
  $effect(() => {
    canFullscreen = document.fullscreenEnabled;
  });

  function resumePlayback(media: HTMLMediaElement): void {
    if (resumeFrom?.projectId !== project.id) resumeFrom = null;
    if (!resumeFrom) return;
    media.currentTime = resumeFrom.seconds;
    if (resumeFrom.playing) void media.play();
    resumeFrom = null;
  }

  function toggleFullscreen(): void {
    if (document.fullscreenElement) void document.exitFullscreen();
    else void monitor?.requestFullscreen();
  }
</script>

<svelte:document
  onfullscreenchange={() =>
    (isFullscreen = document.fullscreenElement === monitor)}
/>

<section
  bind:this={monitor}
  class="flex min-w-0 flex-col bg-surface lg:min-h-0 {isFullscreen
    ? 'h-dvh'
    : ''}"
  aria-label="Subtitle preview"
>
  <div
    class="relative grid place-items-center overflow-hidden {isFullscreen
      ? 'min-h-0 flex-1'
      : 'lg:min-h-0 lg:flex-1'} {isVideo
      ? `bg-black [container-type:size] ${isFullscreen ? '' : 'aspect-video lg:aspect-auto'}`
      : 'bg-surface-sunken py-8'}"
  >
    {#if !project.hasSourceMedia}
      <div class="grid justify-items-center gap-3 text-text-muted">
        <span
          class="grid size-12 place-items-center rounded-full border border-border bg-surface"
          aria-hidden="true"
        >
          <Icon name="play" size={22} />
        </span>
        <p class="text-sm">No source media is attached</p>
      </div>
    {:else if isVideo}
      <!-- Sized to the picture so the cue overlay sits on it rather than on letterboxing. -->
      <div
        class="relative"
        style:aspect-ratio={videoAspectRatio}
        style:width={`min(100cqw, ${videoAspectRatio * 100}cqh)`}
      >
        <!-- The cues being edited are drawn over the picture instead of a caption track. -->
        <!-- svelte-ignore a11y_media_has_caption -->
        <video
          bind:this={transport.mediaElement}
          bind:volume
          bind:muted
          class="size-full object-contain"
          onloadedmetadata={(event) => {
            const media = event.currentTarget;
            if (media.videoWidth > 0 && media.videoHeight > 0) {
              videoAspectRatio = media.videoWidth / media.videoHeight;
            }
            resumePlayback(media);
          }}
          onclick={() => transport.togglePlayback()}
          preload="metadata"
          src={previewUrl}
          poster={thumbnailUrl ?? undefined}
          {...transport.mediaEvents}
          onerror={() => {
            resumeFrom = null;
            transport.mediaEvents.onerror();
          }}
        ></video>
        {#if transport.activeCue}
          <p
            class="pointer-events-none absolute inset-x-[10%] bottom-[8%] text-center leading-relaxed font-semibold text-white {isFullscreen
              ? 'text-2xl lg:text-3xl'
              : 'text-base lg:text-lg'}"
          >
            <span
              class="rounded bg-black/75 px-2 py-0.5 whitespace-pre-line box-decoration-clone"
            >
              {transport.activeCue.text}
            </span>
          </p>
        {/if}
      </div>
    {:else}
      <div class="grid w-full max-w-md justify-items-center gap-3 px-6">
        <span
          class="grid size-14 place-items-center rounded-full border border-border bg-surface text-text-muted"
          aria-hidden="true"
        >
          <Icon name="audio" size={26} />
        </span>
        <p class="max-w-full truncate text-sm font-medium text-text">
          {project.mediaFilename}
        </p>
        <audio
          bind:this={transport.mediaElement}
          bind:volume
          bind:muted
          preload="metadata"
          src={previewUrl}
          onloadedmetadata={(event) => resumePlayback(event.currentTarget)}
          {...transport.mediaEvents}
          onerror={() => {
            resumeFrom = null;
            transport.mediaEvents.onerror();
          }}
        ></audio>
        <p class="min-h-14 text-center text-base font-semibold text-text">
          <span class="whitespace-pre-line">
            {transport.activeCue?.text ?? ""}
          </span>
        </p>
      </div>
    {/if}
  </div>

  <PlayerControls
    {transport}
    bind:volume
    bind:muted
    fullscreen={isVideo && canFullscreen
      ? { active: isFullscreen, toggle: toggleFullscreen }
      : null}
    {audioMenu}
  />
</section>
