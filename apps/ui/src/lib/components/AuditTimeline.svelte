<script lang="ts">
  import type { TimelineEntry } from '$lib/types';
  import { formatTime } from '$lib/utils';

  let { timeline }: { timeline: TimelineEntry[] } = $props();
</script>

<section class="card">
  <h2>Audit Timeline</h2>
  {#if timeline.length === 0}
    <p class="muted">No timeline entries yet.</p>
  {:else}
    <ol class="timeline">
      {#each [...timeline].reverse() as entry, i (i)}
        <li class="timeline-entry">
          <div class="entry-connector">
            <span class="entry-dot dot-{entry.status}"></span>
            {#if i < timeline.length - 1}
              <span class="entry-line"></span>
            {/if}
          </div>
          <div class="entry-body">
            <div class="entry-main">
              <span class="entry-step mono">{entry.step}</span>
              <span class="badge tbadge-{entry.status}">{entry.status}</span>
              <span class="entry-time">{formatTime(entry.timestamp)}</span>
            </div>
            {#if entry.details}
              <p class="entry-details">{entry.details}</p>
            {/if}
            {#if entry.metadata && Object.keys(entry.metadata).length > 0}
              <dl class="entry-meta">
                {#each Object.entries(entry.metadata) as [k, v] (k)}
                  <dt>{k}</dt>
                  <dd class="mono">{v}</dd>
                {/each}
              </dl>
            {/if}
          </div>
        </li>
      {/each}
    </ol>
  {/if}
</section>

<style>
  .timeline {
    list-style: none;
    margin-top: var(--space-1);
  }

  .timeline-entry {
    display: flex;
    gap: var(--space-4);
  }

  .entry-connector {
    display: flex;
    flex-direction: column;
    align-items: center;
    flex-shrink: 0;
    width: 16px;
  }

  .entry-dot {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    flex-shrink: 0;
    margin-top: var(--space-2);
    background: currentColor;
    box-shadow: 0 0 0 3px var(--color-surface);
  }

  .dot-started {
    color: var(--color-blue);
  }

  .dot-completed {
    color: var(--color-green);
  }

  .dot-failed {
    color: var(--color-red);
  }

  .dot-waiting {
    color: var(--color-amber);
  }

  .entry-line {
    flex: 1;
    width: 2px;
    background: var(--color-line);
    margin: var(--space-2) 0 0;
    min-height: var(--space-5);
    border-radius: var(--radius-pill);
  }

  .entry-body {
    flex: 1;
    padding-bottom: var(--space-5);
    min-width: 0;
  }

  .entry-main {
    display: flex;
    align-items: center;
    gap: var(--space-3);
    flex-wrap: wrap;
  }

  .entry-step {
    font-weight: 600;
    color: var(--color-ink);
    font-size: 14px;
  }

  .entry-time {
    font-size: 12px;
    color: var(--color-ink-muted);
    margin-left: auto;
    font-family: var(--font-mono);
    letter-spacing: -0.01em;
  }

  .entry-details {
    font-size: 14px;
    color: var(--color-ink-soft);
    margin-top: var(--space-2);
    line-height: 1.5;
  }

  .entry-meta {
    display: grid;
    grid-template-columns: auto 1fr;
    column-gap: var(--space-4);
    row-gap: 0;
    margin-top: var(--space-3);
    padding: var(--space-3) var(--space-4);
    background: var(--color-surface-alt);
    border: 1px solid var(--color-line-soft);
    border-radius: var(--radius-sm);
    font-size: 13px;
  }

  .entry-meta dt {
    color: var(--color-ink-muted);
    white-space: nowrap;
    padding: var(--space-1) 0;
    font-size: 12px;
    font-weight: 500;
  }

  .entry-meta dd {
    color: var(--color-ink);
    word-break: break-all;
    padding: var(--space-1) 0;
  }

  /* Timeline step status badge colours */
  .tbadge-started {
    background: var(--color-blue-tint);
    color: var(--color-blue);
    border-color: var(--color-blue-line);
  }

  .tbadge-completed {
    background: var(--color-green-pale);
    color: var(--color-green-darker);
    border-color: var(--color-green);
  }

  .tbadge-failed {
    background: var(--color-red-tint);
    color: var(--color-red);
    border-color: var(--color-red-line);
  }

  .tbadge-waiting {
    background: var(--color-amber-tint);
    color: var(--color-amber);
    border-color: var(--color-amber-line);
  }
</style>
