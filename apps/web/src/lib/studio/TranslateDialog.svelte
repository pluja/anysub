<script lang="ts">
  import Button from "#lib/components/ui/Button.svelte";
  import Dialog from "#lib/components/ui/Dialog.svelte";
  import Picker from "#lib/components/ui/Picker.svelte";
  import { languageName } from "#lib/shared/languages.ts";
  import type { SubtitleTrackSummary } from "#lib/shared/subtitles.ts";
  import type {
    DetectedWorkerHardware,
    ModelChoice,
  } from "#lib/shared/transcription-models.ts";
  import { MAX_TARGET_LANGUAGES } from "#lib/shared/translation.ts";
  import ModelPicker from "./ModelPicker.svelte";
  import TargetLanguages from "./TargetLanguages.svelte";
  import TranscriptLabel from "./TranscriptLabel.svelte";
  import {
    currentTranslationLanguages,
    isOriginal,
    lastTranslationModelKey,
    translationSourceFor,
  } from "./transcripts.ts";

  type Props = {
    open?: boolean;
    tracks: SubtitleTrackSummary[];
    showAudioTrack: boolean;
    models: ModelChoice[];
    hardware: DetectedWorkerHardware[];
    canSetUp: boolean;
    onbeforesubmit: () => Promise<string | null>;
  };

  let {
    open = $bindable(false),
    tracks,
    showAudioTrack,
    models,
    hardware,
    canSetUp,
    onbeforesubmit,
  }: Props = $props();

  const id = $props.id();
  let sourceId = $state("");
  let languages = $state<string[]>([]);
  let pickedModelKey = $state("");
  let submitting = $state(false);
  let blockedMessage = $state<string | null>(null);
  const originals = $derived(tracks.filter(isOriginal));
  const source = $derived(originals.find((track) => track.id === sourceId));
  const translated = $derived(
    source ? currentTranslationLanguages(tracks, source) : [],
  );
  const lastModelKey = $derived(lastTranslationModelKey(tracks));
  const model = $derived(
    models.find((candidate) => candidate.key === pickedModelKey) ??
      models.find((candidate) => candidate.key === lastModelKey) ??
      models[0],
  );
  const modelAdvice = $derived(
    models.some((candidate) => candidate.remote) &&
      models.some((candidate) => !candidate.remote)
      ? "A provider's chat model sounds more natural. The local model keeps the text on this server."
      : models.some((candidate) => candidate.remote)
        ? "Runs on your provider's servers."
        : "Runs on this server, so the text never leaves it.",
  );

  // Opening on a translation starts from the original it came from.
  export function show(
    trackId: string | null,
    chosenLanguages: string[] = [],
  ): void {
    const start = translationSourceFor(tracks, trackId ?? "");
    sourceId = start?.id ?? "";
    languages = chosenLanguages.filter((code) => code !== start?.language);
    blockedMessage = null;
    submitting = false;
    open = true;
  }

  // A transcript cannot be translated into its own language.
  function pickSource(trackId: string): void {
    sourceId = trackId;
    const language = tracks.find((track) => track.id === trackId)?.language;
    languages = languages.filter((code) => code !== language);
  }

  async function handleSubmit(event: SubmitEvent): Promise<void> {
    event.preventDefault();
    const form = event.currentTarget as HTMLFormElement;
    submitting = true;
    blockedMessage = await onbeforesubmit();
    if (blockedMessage) {
      submitting = false;
      return;
    }
    form.submit();
  }
</script>

<Dialog
  bind:open
  title="Translate transcript"
  description="Each language becomes a new transcript with the original's timing. Cues that run long are split."
>
  {#if models.length === 0}
    <div class="grid gap-2 text-sm text-text-muted">
      <p>Translation is not set up on this server yet.</p>
      {#if canSetUp}
        <p>
          Choose a provider's chat model, or download the local model, in
          <a
            href="/admin/translation"
            class="font-medium text-accent underline-offset-2 hover:underline"
          >
            Administration, Translation
          </a>.
        </p>
      {:else}
        <p>Ask an administrator to set it up.</p>
      {/if}
    </div>
  {:else if source && model}
    <form
      id="{id}-form"
      method="POST"
      action="?/translate"
      class="grid gap-5"
      onsubmit={handleSubmit}
    >
      {#if originals.length > 1}
        <Picker
          label="Translate from"
          name="sourceTrackId"
          options={originals}
          key={(track) => track.id}
          optionText={(track) => languageName(track.language)}
          bind:value={() => sourceId, pickSource}
          title="Choose a transcript"
          divided
        >
          {#snippet selection(track)}
            {#if track}<TranscriptLabel {track} {showAudioTrack} />{/if}
          {/snippet}
          {#snippet option(track)}
            <TranscriptLabel {track} {showAudioTrack} />
          {/snippet}
        </Picker>
      {:else}
        <div class="grid min-w-0 gap-1.5">
          <span class="text-sm font-medium text-text">Translate from</span>
          <input type="hidden" name="sourceTrackId" value={source.id} />
          <div
            class="flex rounded-md border border-border bg-surface-sunken px-3 py-2"
          >
            <TranscriptLabel track={source} {showAudioTrack} />
          </div>
        </div>
      {/if}
      <TargetLanguages
        bind:value={languages}
        excluded={[source.language]}
        {translated}
        max={MAX_TARGET_LANGUAGES}
      />
      <ModelPicker
        {models}
        {hardware}
        selectedKey={model.key}
        onpick={(key) => (pickedModelKey = key)}
        subtitle={modelAdvice}
      />
      {#if blockedMessage}
        <p class="text-sm text-danger" role="alert">{blockedMessage}</p>
      {/if}
    </form>
  {/if}

  {#snippet actions()}
    <Button variant="ghost" onclick={() => (open = false)}>
      {models.length === 0 ? "Close" : "Cancel"}
    </Button>
    {#if models.length > 0 && source}
      <Button
        type="submit"
        form="{id}-form"
        variant="primary"
        loading={submitting}
        disabled={languages.length === 0}
      >
        {languages.length > 1
          ? `Translate into ${languages.length} languages`
          : "Translate"}
      </Button>
    {/if}
  {/snippet}
</Dialog>
