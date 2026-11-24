<script lang="ts" module>
  import type { AvailableTranscriptionModel } from "#lib/shared/transcription-models.ts";

  export function processingNote(
    models: AvailableTranscriptionModel[],
  ): string {
    return models.some((model) => model.remote)
      ? "Local models run on your worker. A provider's model sends the audio to that provider."
      : "Processing stays on your connected worker. No media leaves Anysub.";
  }
</script>

<script lang="ts">
  import type { Snippet } from "svelte";
  import Icon from "#lib/components/Icon.svelte";
  import Button from "#lib/components/ui/Button.svelte";
  import Field from "#lib/components/ui/Field.svelte";
  import Input from "#lib/components/ui/Input.svelte";
  import { CONTROL_CLASSES } from "#lib/components/ui/variants.ts";
  import {
    transcriptionLanguageOf,
    type AudioTrack,
  } from "#lib/shared/audio-tracks.ts";
  import {
    MAX_LINE_LENGTH,
    MIN_LINE_LENGTH,
    type TranscriptionDefaults,
  } from "#lib/shared/instance-settings.ts";
  import { DETECT_LANGUAGE, languageName } from "#lib/shared/languages.ts";
  import { MAX_BATCH_SIZE, MAX_PROMPT_LENGTH } from "#lib/shared/limits.ts";
  import { formatDuration } from "#lib/shared/format.ts";
  import {
    formatPricePerMinute,
    formatUsd,
    priceForDuration,
  } from "#lib/shared/money.ts";
  import type { SubtitleTrackSummary } from "#lib/shared/subtitles.ts";
  import {
    acceptsOption,
    defaultModelKey,
    type DetectedWorkerHardware,
  } from "#lib/shared/transcription-models.ts";
  import AudioTrackPicker from "./AudioTrackPicker.svelte";
  import LanguagePicker from "./LanguagePicker.svelte";
  import ModelPicker from "./ModelPicker.svelte";
  import { matchingTranscript } from "./transcripts.ts";

  type Props = {
    models: AvailableTranscriptionModel[];
    hardware: DetectedWorkerHardware[];
    defaults: TranscriptionDefaults;
    mediaDurationUs?: number | null;
    audioTracks?: AudioTrack[];
    // The track the studio plays, which a new transcript starts on.
    audioTrack?: number;
    // The project's transcripts, to point out one this run would repeat.
    transcripts?: SubtitleTrackSummary[];
    // Resolves to an error message when the transcription must not start.
    onbeforesubmit?: () => Promise<string | null>;
    intro?: Snippet;
  };

  let {
    models,
    hardware,
    defaults,
    mediaDurationUs = null,
    audioTracks = [],
    audioTrack = 0,
    transcripts = [],
    onbeforesubmit,
    intro,
  }: Props = $props();

  // In a side panel the options scroll and the start button stays pinned to
  // the panel's bottom. A dialog scrolls as a whole, so the button sticks.
  const BODY_CLASSES =
    "min-h-0 flex-1 overflow-y-auto px-4 pt-4 in-[dialog]:overflow-visible in-[dialog]:p-0";
  const COLUMN_CLASSES = "mx-auto grid w-full max-w-md gap-4";

  let pickedModelKey = $state<string>();
  const defaultKey = $derived(defaultModelKey(models, defaults.model));
  const selectedModel = $derived(
    models.find((model) => model.key === pickedModelKey) ??
      models.find((model) => model.key === defaultKey),
  );
  const accepts = (option: string) =>
    selectedModel ? acceptsOption(selectedModel.optionsSchema, option) : true;
  const advancedOptionsSummary = $derived.by(() => {
    const names = [
      accepts("alignment") ? "word alignment" : null,
      accepts("prompt") ? "prompt" : null,
      "line length",
      accepts("batchSize") ? "batch size" : null,
    ].filter((name) => name !== null);
    const list = new Intl.ListFormat("en-GB").format(names);
    return list.charAt(0).toUpperCase() + list.slice(1);
  });
  const remote = $derived(selectedModel?.remote ?? null);
  const estimatedCost = $derived(
    remote?.priceMicrosPerMinute != null && mediaDurationUs
      ? priceForDuration(remote.priceMicrosPerMinute, mediaDurationUs)
      : null,
  );
  // Held here so the choice survives picking a remote model, which hides it.
  // svelte-ignore state_referenced_locally
  let alignment = $state(defaults.alignment);
  // svelte-ignore state_referenced_locally
  let language = $state(defaults.language ?? DETECT_LANGUAGE);
  // svelte-ignore state_referenced_locally
  let pickedAudioTrack = $state(audioTrack);
  const repeated = $derived(
    selectedModel &&
      matchingTranscript(transcripts, {
        modelKey: selectedModel.key,
        language: language === DETECT_LANGUAGE ? null : language,
        audioTrack: pickedAudioTrack,
      }),
  );

  // A track tagged with a language Whisper knows sets the spoken language.
  function pickAudioTrack(index: number): void {
    pickedAudioTrack = index;
    const track = audioTracks.find((candidate) => candidate.index === index);
    const trackLanguage = track ? transcriptionLanguageOf(track) : null;
    if (trackLanguage) language = trackLanguage;
  }

  let preparing = $state(false);
  let blockedMessage = $state<string | null>(null);

  function revealAdvancedOptions(
    event: Event & { currentTarget: HTMLDetailsElement },
  ): void {
    if (event.currentTarget.open) {
      event.currentTarget.lastElementChild?.scrollIntoView({
        block: "nearest",
      });
    }
  }

  async function handleSubmit(event: SubmitEvent): Promise<void> {
    if (!onbeforesubmit) return;
    event.preventDefault();
    const form = event.currentTarget as HTMLFormElement;
    preparing = true;
    blockedMessage = await onbeforesubmit();
    if (blockedMessage) {
      preparing = false;
      return;
    }
    form.submit();
  }
</script>

{#if models.length}
  <form
    class="flex min-h-0 flex-1 flex-col"
    method="POST"
    action="?/transcribe"
    onsubmit={handleSubmit}
  >
    <div class={BODY_CLASSES}>
      <div class="{COLUMN_CLASSES} pb-4">
        {@render intro?.()}
        {#if audioTracks.length > 1}
          <AudioTrackPicker
            tracks={audioTracks}
            bind:value={() => pickedAudioTrack, pickAudioTrack}
          />
        {/if}
        <ModelPicker
          {models}
          {hardware}
          selectedKey={selectedModel?.key ?? ""}
          onpick={(key) => (pickedModelKey = key)}
        />
        <LanguagePicker bind:value={language} />
        <details
          class="group rounded-lg border border-border"
          ontoggle={revealAdvancedOptions}
        >
          <summary
            class="flex cursor-pointer list-none items-center gap-3 rounded-lg px-3.5 py-3 transition-colors duration-150 group-open:rounded-b-none hover:bg-surface-sunken focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-focus motion-reduce:transition-none [&::-webkit-details-marker]:hidden"
          >
            <span class="grid min-w-0 flex-1 gap-0.5">
              <span class="text-sm font-medium text-text">
                Advanced options
              </span>
              <span class="text-xs text-text-muted">
                {advancedOptionsSummary}
              </span>
            </span>
            <span
              class="grid size-4 shrink-0 place-items-center text-text-muted transition-transform duration-150 group-open:rotate-180 motion-reduce:transition-none"
              aria-hidden="true"
            >
              <Icon name="chevron-down" size={16} />
            </span>
          </summary>
          <!-- The bottom scroll margin clears the pinned start button. -->
          <div
            class="grid scroll-mb-18 grid-cols-2 items-start gap-x-3 gap-y-4 border-t border-border p-3.5"
          >
            {#if accepts("alignment")}
              <label class="col-span-2 flex cursor-pointer items-start gap-2.5">
                <input
                  class="mt-0.5 size-4 shrink-0 accent-accent focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-focus"
                  name="alignment"
                  type="checkbox"
                  bind:checked={alignment}
                />
                <span class="grid gap-0.5">
                  <span class="text-sm font-medium text-text">
                    Word alignment
                  </span>
                  <span class="text-sm text-text-muted">
                    Word-level timestamps, for editing and karaoke.
                    Transcription is faster if off.
                  </span>
                </span>
              </label>
            {/if}
            {#if accepts("prompt")}
              <div class="col-span-2">
                <Field
                  label="Prompt"
                  hint="A sentence with names or terms to spell right, in the spoken language."
                >
                  {#snippet control({ id, describedBy })}
                    <textarea
                      {id}
                      aria-describedby={describedBy}
                      class="{CONTROL_CLASSES} resize-y py-2"
                      name="prompt"
                      rows="2"
                      maxlength={MAX_PROMPT_LENGTH}
                      placeholder="Dr. Okonkwo talks about Anysub and WhisperX."
                    ></textarea>
                  {/snippet}
                </Field>
              </div>
            {/if}
            <Field label="Line length" hint="Characters per line.">
              {#snippet control({ id, describedBy })}
                <Input
                  {id}
                  aria-describedby={describedBy}
                  name="maxLineLength"
                  type="number"
                  min={MIN_LINE_LENGTH}
                  max={MAX_LINE_LENGTH}
                  value={defaults.maxLineLength}
                  required
                />
              {/snippet}
            </Field>
            {#if accepts("batchSize")}
              <Field label="Batch size" hint="Lower it if memory runs out.">
                {#snippet control({ id, describedBy })}
                  <Input
                    {id}
                    aria-describedby={describedBy}
                    name="batchSize"
                    type="number"
                    min="1"
                    max={MAX_BATCH_SIZE}
                    placeholder="Automatic"
                  />
                {/snippet}
              </Field>
            {/if}
          </div>
        </details>
      </div>
    </div>
    <div
      class="sticky bottom-0 shrink-0 border-t border-border bg-surface p-4 in-[dialog]:bg-surface-raised in-[dialog]:px-0 in-[dialog]:pb-0"
    >
      <div class="mx-auto grid w-full max-w-md gap-2">
        {#if remote}
          <p class="flex items-start gap-1.5 text-xs text-text-muted">
            <span class="mt-px shrink-0" aria-hidden="true">
              <Icon name="globe" size={14} />
            </span>
            <span>
              Sends the audio to {remote.providerName} at {remote.host}.
              {#if estimatedCost !== null && mediaDurationUs !== null}
                About {formatUsd(estimatedCost)} for {formatDuration(
                  mediaDurationUs,
                )}.
              {:else if remote.priceMicrosPerMinute != null}
                {formatPricePerMinute(remote.priceMicrosPerMinute)}.
              {/if}
            </span>
          </p>
        {/if}
        {#if repeated}
          <p class="flex items-start gap-1.5 text-xs text-text-muted">
            <span class="mt-px shrink-0" aria-hidden="true">
              <Icon name="copy" size={14} />
            </span>
            <span>
              This project already has a transcript in {languageName(
                repeated.language,
              )} from this model. With the same settings, a new one usually comes
              out the same.
            </span>
          </p>
        {/if}
        {#if blockedMessage}
          <p class="text-sm text-danger" role="alert">{blockedMessage}</p>
        {/if}
        <Button
          variant="primary"
          type="submit"
          class="w-full"
          loading={preparing}
        >
          <Icon name="activity" size={16} />
          Start transcription
        </Button>
      </div>
    </div>
  </form>
{:else}
  <div class={BODY_CLASSES}>
    <div class="{COLUMN_CLASSES} pb-4">
      {@render intro?.()}
      <div class="grid gap-3">
        <p class="text-sm text-text-muted">
          A transcription worker is not connected yet. Start or inspect the
          worker container, then this page will offer its models.
        </p>
        <Button class="w-full" disabled>Waiting for a speech worker</Button>
        <p class="text-xs text-text-muted">
          Model choices appear automatically.
        </p>
      </div>
    </div>
  </div>
{/if}
