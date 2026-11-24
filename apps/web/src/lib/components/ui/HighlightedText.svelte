<script lang="ts">
  import type { TextSegment } from "#lib/shared/search-query.ts";

  let { segments }: { segments: TextSegment[] } = $props();
</script>

{#each segments as segment, index (index)}{#if segment.match}<mark
      class="rounded-[3px] bg-highlight px-px text-text"
    >
      {segment.text}
    </mark>{:else}{segment.text}{/if}{/each}
