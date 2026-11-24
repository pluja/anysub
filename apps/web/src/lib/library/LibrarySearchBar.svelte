<script lang="ts">
  import { goto } from "$app/navigation";
  import Icon from "#lib/components/Icon.svelte";
  import Input from "#lib/components/ui/Input.svelte";
  import { buttonClasses } from "#lib/components/ui/variants.ts";
  import { isTextEntryTarget } from "#lib/browser/text-entry.ts";
  import { languageFlag, languageName } from "#lib/shared/languages.ts";
  import type {
    LibraryAdded,
    LibrarySearch,
    LibrarySort,
    LibraryStatus,
  } from "#lib/shared/library-data.ts";
  import { PROJECT_SEARCH_MAX_LENGTH } from "#lib/shared/limits.ts";
  import FilterMenu from "./FilterMenu.svelte";
  import { defaultSort, libraryUrl } from "./search-url";

  type Props = { search: LibrarySearch; languages: string[] };

  let { search, languages }: Props = $props();

  // Long enough to skip the searches in between while someone types.
  const TYPING_PAUSE_MS = 250;

  let input = $state<HTMLInputElement>();
  // svelte-ignore state_referenced_locally
  let draft = $state(search.query);
  let typingTimer: number | undefined;
  let hasFilters = $derived(
    search.language !== null || search.status !== null || search.added !== null,
  );

  // Back and forward change the query while the box keeps what was typed.
  $effect(() => {
    const query = search.query;
    if (document.activeElement !== input) draft = query;
  });

  $effect(() => () => window.clearTimeout(typingTimer));

  function show(changes: Partial<LibrarySearch>, replace = false) {
    const query = changes.query ?? search.query;
    // A sort nobody chose follows the query: best match while searching.
    const sort =
      changes.sort ??
      (search.sort === defaultSort(search.query)
        ? defaultSort(query)
        : search.sort);
    void goto(libraryUrl({ ...search, ...changes, query, sort }), {
      reset: false,
      replace,
    });
  }

  function handleInput() {
    window.clearTimeout(typingTimer);
    typingTimer = window.setTimeout(
      () => show({ query: draft.trim() }, true),
      TYPING_PAUSE_MS,
    );
  }

  function submitNow(event: SubmitEvent) {
    event.preventDefault();
    window.clearTimeout(typingTimer);
    show({ query: draft.trim() });
  }

  function clearQuery() {
    window.clearTimeout(typingTimer);
    draft = "";
    show({ query: "" });
    input?.focus();
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === "Escape" && draft) {
      event.preventDefault();
      clearQuery();
    }
  }

  function focusOnSlash(event: KeyboardEvent) {
    if (
      event.key !== "/" ||
      event.ctrlKey ||
      event.metaKey ||
      event.altKey ||
      isTextEntryTarget(event.target)
    ) {
      return;
    }
    event.preventDefault();
    input?.focus();
    input?.select();
  }

  const STATUS_OPTIONS: { value: LibraryStatus | null; label: string }[] = [
    { value: null, label: "Any status" },
    { value: "ready", label: "Ready" },
    { value: "active", label: "In progress" },
    { value: "attention", label: "Needs attention" },
  ];
  const ADDED_OPTIONS: { value: LibraryAdded | null; label: string }[] = [
    { value: null, label: "Any time" },
    { value: "week", label: "Past week" },
    { value: "month", label: "Past month" },
    { value: "year", label: "Past year" },
  ];
  let sortOptions = $derived<{ value: LibrarySort; label: string }[]>([
    ...(search.query
      ? [{ value: "relevance" as const, label: "Best match" }]
      : []),
    { value: "activity", label: "Last edited" },
    { value: "added", label: "Date added" },
    { value: "name", label: "Name" },
  ]);
  let languageOptions = $derived([
    { value: null, label: "Any language", flag: null },
    ...languages.map((code) => ({
      value: code,
      label: languageName(code),
      flag: languageFlag(code),
    })),
  ]);
</script>

<svelte:window onkeydown={focusOnSlash} />

<div class="mt-6 grid gap-3">
  <form
    method="GET"
    action="/"
    role="search"
    class="relative max-w-2xl"
    onsubmit={submitNow}
  >
    <label for="library-search" class="sr-only">
      Search project names and transcripts
    </label>
    <span
      class="pointer-events-none absolute top-1/2 left-3 -translate-y-1/2 text-text-subtle"
      aria-hidden="true"
    >
      <Icon name="search" size={16} />
    </span>
    <Input
      bind:ref={input}
      id="library-search"
      type="search"
      name="q"
      autocomplete="off"
      spellcheck="false"
      bind:value={draft}
      oninput={handleInput}
      onkeydown={handleKeydown}
      placeholder="Search names and transcripts"
      maxlength={PROJECT_SEARCH_MAX_LENGTH}
      class="pr-10 pl-9 [&::-webkit-search-cancel-button]:appearance-none"
    />
    {#each [["language", search.language], ["status", search.status], ["added", search.added]] as [name, value] (name)}
      {#if value}<input type="hidden" {name} {value} />{/if}
    {/each}
    {#if draft}
      <button
        type="button"
        aria-label="Clear search"
        title="Clear search"
        onclick={clearQuery}
        class="{buttonClasses(
          'ghost',
          'sm',
          true,
        )} absolute top-1/2 right-1 size-8 -translate-y-1/2"
      >
        <Icon name="close" size={14} />
      </button>
    {:else}
      <kbd
        class="pointer-events-none absolute top-1/2 right-3 -translate-y-1/2 rounded border border-border px-1.5 font-mono text-xs text-text-subtle max-sm:hidden"
        title="Press / to search"
        aria-hidden="true"
      >
        /
      </kbd>
    {/if}
  </form>

  <div class="flex flex-wrap items-center gap-2">
    {#if languages.length > 1}
      <FilterMenu
        label="Language"
        options={languageOptions}
        selected={search.language}
        onchoose={(language) => show({ language })}
      />
    {/if}
    <FilterMenu
      label="Status"
      options={STATUS_OPTIONS}
      selected={search.status}
      onchoose={(status) => show({ status })}
    />
    <FilterMenu
      label="Added"
      options={ADDED_OPTIONS}
      selected={search.added}
      onchoose={(added) => show({ added })}
    />
    {#if hasFilters}
      <button
        type="button"
        class="rounded-sm px-1 text-sm text-accent hover:underline focus-visible:outline-2 focus-visible:outline-focus"
        onclick={() => show({ language: null, status: null, added: null })}
      >
        Clear filters
      </button>
    {/if}
    <div class="ml-auto">
      <FilterMenu
        label="Sort"
        isSort
        align="end"
        options={sortOptions}
        selected={search.sort === "relevance" && !search.query
          ? "activity"
          : search.sort}
        onchoose={(sort) => sort && show({ sort })}
      />
    </div>
  </div>
</div>
