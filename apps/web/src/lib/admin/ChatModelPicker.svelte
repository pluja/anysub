<script lang="ts">
  import Field from "#lib/components/ui/Field.svelte";
  import Input from "#lib/components/ui/Input.svelte";
  import Picker from "#lib/components/ui/Picker.svelte";
  import { matchesSearch } from "#lib/shared/search.ts";

  type Props = {
    providerId: string;
    providerName: string;
    // The saved model, kept while it belongs to this provider.
    initial: string;
    error?: string;
  };

  type ModelList =
    | { state: "loading" }
    | { state: "listed"; models: string[] }
    // The provider has no list, or it could not be read: the ID is typed.
    | { state: "unlisted"; problem: string | null };

  let { providerId, providerName, initial, error }: Props = $props();

  let list = $state<ModelList>({ state: "loading" });
  // svelte-ignore state_referenced_locally
  let value = $state(initial);

  $effect(() => {
    const requested = providerId;
    list = { state: "loading" };
    void loadModels(requested);
  });

  const typedHint = $derived(
    list.state === "listed"
      ? `None of ${providerName}'s models looks like a chat model. Enter the model's ID.`
      : list.state === "unlisted" && list.problem
        ? `${list.problem} Enter the model's ID instead.`
        : `${providerName} does not list its models. Enter the model's ID, such as llama3.3.`,
  );

  async function loadModels(requested: string): Promise<void> {
    let next: ModelList;
    try {
      const response = await fetch(
        `/admin/translation/chat-models?providerId=${encodeURIComponent(requested)}`,
        { cache: "no-store" },
      );
      const body: { models?: string[] | null; message?: string } =
        await response.json();
      next = !response.ok
        ? { state: "unlisted", problem: body.message ?? null }
        : body.models
          ? { state: "listed", models: body.models }
          : { state: "unlisted", problem: null };
    } catch {
      next = {
        state: "unlisted",
        problem: `Could not ask ${providerName} for its models.`,
      };
    }
    if (requested !== providerId) return;
    list = next;
    if (next.state === "listed" && value && !next.models.includes(value)) {
      value = "";
    }
  }
</script>

{#if list.state === "loading"}
  <div class="grid gap-1.5">
    <span class="text-sm font-medium text-text">Chat model</span>
    <p
      class="flex h-9 items-center gap-2 rounded-md border border-border px-3 text-sm text-text-muted"
      role="status"
    >
      <span
        class="size-3 animate-spin rounded-full border-2 border-current border-t-transparent motion-reduce:animate-none"
        aria-hidden="true"
      ></span>
      Loading {providerName}'s models…
    </p>
  </div>
{:else if list.state === "listed" && list.models.length > 0}
  <div class="grid gap-1.5">
    <Picker
      label="Chat model"
      name="model"
      options={list.models}
      key={(model) => model}
      bind:value
      title="Choose a chat model"
      subtitle="{list.models.length} chat models at {providerName}"
      searchLabel="Search models"
      matches={(model, query) => matchesSearch(query, [model])}
    >
      {#snippet selection(model)}
        {#if model}
          <code class="font-mono text-sm text-text">{model}</code>
        {:else}
          <span class="text-sm text-text-muted">Choose a model</span>
        {/if}
      {/snippet}
      {#snippet option(model)}
        <code class="min-w-0 truncate font-mono text-sm text-text">
          {model}
        </code>
      {/snippet}
    </Picker>
    {#if error}
      <p class="text-sm text-danger" role="alert">{error}</p>
    {:else}
      <p class="text-sm text-text-muted">
        Larger models translate idioms and tone better.
      </p>
    {/if}
  </div>
{:else}
  <Field label="Chat model" hint={typedHint} {error}>
    {#snippet control({ id, describedBy, invalid })}
      <Input
        {id}
        name="model"
        bind:value
        required
        spellcheck="false"
        autocomplete="off"
        aria-describedby={describedBy}
        aria-invalid={invalid}
      />
    {/snippet}
  </Field>
{/if}
