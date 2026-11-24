<script lang="ts">
  import { enhance } from "$app/forms";
  import ChatModelPicker from "#lib/admin/ChatModelPicker.svelte";
  import FormMessage from "#lib/admin/FormMessage.svelte";
  import { FormSubmission } from "#lib/admin/form-submission.svelte.ts";
  import SectionHeader from "#lib/admin/SectionHeader.svelte";
  import SettingsCard from "#lib/admin/SettingsCard.svelte";
  import WorkerModels from "#lib/admin/WorkerModels.svelte";
  import Icon from "#lib/components/Icon.svelte";
  import ProviderIcon from "#lib/components/provider-icons/ProviderIcon.svelte";
  import Button from "#lib/components/ui/Button.svelte";
  import Picker from "#lib/components/ui/Picker.svelte";

  let { data, form } = $props();

  type ProviderOption = (typeof data.providers)[number];
  const NO_PROVIDER: ProviderOption = {
    id: "",
    name: "None",
    brand: null,
    host: "Translate only with the local model",
  };

  const submission = new FormSubmission();
  // svelte-ignore state_referenced_locally
  let providerId = $state(data.providerId ?? "");
  const provider = $derived(
    data.providers.find((candidate) => candidate.id === providerId),
  );
  const problem = $derived(
    form && "problem" in form ? (form.problem ?? null) : null,
  );
  const saved = $derived(form && "saved" in form && form.saved);
</script>

<SectionHeader
  title="Translation"
  description="People translate a transcript from the studio, into one or more languages at a time. Each translation becomes a new transcript they can edit and export."
/>

{#if !data.workerConnected}
  <p
    class="mb-4 flex items-start gap-2 rounded-md bg-warning-soft px-3 py-2 text-sm text-warning"
  >
    <span class="mt-0.5 shrink-0"><Icon name="alert" size={16} /></span>
    No connected worker can translate yet. Start the worker, or update it to this
    version of Anysub.
  </p>
{/if}

<FormMessage {form} />

{#snippet providerLabel(option: ProviderOption)}
  <span class="flex min-w-0 items-center gap-2.5">
    {#if option.id}
      <ProviderIcon name={option.name} brand={option.brand} size={18} />
    {:else}
      <span class="shrink-0 text-text-muted" aria-hidden="true">
        <Icon name="cpu" size={18} />
      </span>
    {/if}
    <span class="min-w-0 truncate text-sm font-medium text-text">
      {option.name}
    </span>
    <span class="min-w-0 truncate text-xs text-text-muted">{option.host}</span>
  </span>
{/snippet}

<div class="grid gap-4">
  <SettingsCard
    icon="globe"
    title="Provider model"
    description="A chat model from one of your external providers gives the most natural subtitles. When one is set, it is the default."
  >
    {#if data.providers.length === 0}
      <p class="text-sm text-text-muted">
        Add a provider such as OpenAI, OpenRouter or your own server in
        <a
          href="/admin/providers"
          class="font-medium text-accent underline-offset-2 hover:underline"
        >
          External providers
        </a>, then choose one of its chat models here.
      </p>
    {:else}
      <form
        method="POST"
        action="?/save"
        class="grid gap-4"
        use:enhance={submission.handle()}
      >
        <div class="grid gap-1.5">
          <Picker
            label="Provider"
            name="providerId"
            options={[...data.providers, NO_PROVIDER]}
            key={(option) => option.id}
            optionText={(option) => option.name}
            bind:value={providerId}
            title="Choose a provider"
            divided
          >
            {#snippet selection(option)}
              {#if option}{@render providerLabel(option)}{/if}
            {/snippet}
            {#snippet option(option)}
              {@render providerLabel(option)}
            {/snippet}
          </Picker>
          {#if problem?.field === "providerId"}
            <p class="text-sm text-danger" role="alert">{problem.message}</p>
          {/if}
        </div>
        {#if provider}
          {#key provider.id}
            <ChatModelPicker
              providerId={provider.id}
              providerName={provider.name}
              initial={data.providerId === provider.id
                ? (data.model ?? "")
                : ""}
              error={problem?.field === "model" ? problem.message : undefined}
            />
          {/key}
          <p
            class="flex items-start gap-2 rounded-md bg-surface-sunken px-3 py-2 text-sm text-text-muted"
          >
            <span class="mt-0.5 shrink-0"><Icon name="globe" size={16} /></span>
            <span>
              Translating with it sends the transcript's text to {provider.host},
              which may keep it and charge for it under its own terms.
            </span>
          </p>
        {/if}
        <div class="flex flex-wrap items-center justify-end gap-3">
          <p aria-live="polite" class="mr-auto text-sm">
            {#if saved}
              <span class="inline-flex items-center gap-1.5 text-success">
                <Icon name="check-circle" size={16} />
                Saved. New translations use this model.
              </span>
            {/if}
          </p>
          <Button type="submit" variant="primary" loading={submission.busy}>
            Save
          </Button>
        </div>
      </form>
    {/if}
  </SettingsCard>

  <SettingsCard
    icon="cube"
    title="Local model"
    description="Translates on this server, so transcripts never leave it. It is more literal than a chat model, so review its subtitles before publishing. Once downloaded, it is offered in the studio."
  >
    {#if data.workers.length > 0}
      <WorkerModels workers={data.workers} label="Translation models" />
    {:else}
      <p class="text-sm text-text-muted">
        No connected worker can run a local translation model.
      </p>
    {/if}
    <p class="text-xs text-text-muted">Models download from Hugging Face.</p>
  </SettingsCard>
</div>
