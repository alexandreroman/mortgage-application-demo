<script lang="ts">
  import type { MortgageApplication } from '$lib/types';
  import {
    formatTime,
    isNonRunningTerminal,
    lifecycleLabel,
    statusLabel,
    workflowVersionLabel,
    workflowVersionStyle,
  } from '$lib/utils';

  let {
    app,
    isTerminal,
    refreshing,
    refreshError,
    onRefresh,
    refreshTimeout,
  }: {
    app: MortgageApplication;
    isTerminal: boolean;
    refreshing: boolean;
    refreshError: string;
    onRefresh: () => void;
    refreshTimeout: number;
  } = $props();

  let now = $state(Date.now());

  // Workflows in TERMINATED, CANCELLED, TIMED_OUT or FAILED have stopped
  // running without giving the workflow a chance to clear transient SLA
  // fields, so a query response can still carry pendingDependency /
  // slaDeadline from the moment of the stop. Treat all such lifecycle
  // states as "stopped": no live SLA timer, no progress bar, and a neutral
  // per-status badge that does not imply a business outcome.
  //
  // Live SLA logic only fires when the workflow is RUNNING and a
  // dependency is pending. COMPLETED is handled by the existing
  // post-completion path (persisted slaStatus/slaBreached). Undefined
  // workflowStatus (older payloads or pre-fetch state) is treated as
  // running so behaviour is unchanged.
  const stoppedStatus = $derived(
    isNonRunningTerminal(app.workflowStatus) ? app.workflowStatus : undefined,
  );

  // Tick once a second so the elapsed-waiting label updates between API
  // refreshes. Only runs while a dependency is pending, the business state
  // is non-terminal, and the workflow is still actually running.
  $effect(() => {
    if (isTerminal || stoppedStatus !== undefined || !app.pendingSince) return;
    const id = setInterval(() => (now = Date.now()), 1000);
    return () => clearInterval(id);
  });

  // While the workflow is waiting (pendingDependency is set) the SLA row uses
  // live client-side computation against the deadline. Once the wait resolves
  // the workflow persists slaStatus / slaBreached and the row switches to
  // showing the durable outcome without a progress bar or ticking timer.
  // A stopped workflow is never considered "waiting" even if its last query
  // snapshot still carries a pendingDependency.
  const isWaiting = $derived(
    !!app.pendingDependency && stoppedStatus === undefined,
  );

  const pendingSinceMs = $derived(
    app.pendingSince ? new Date(app.pendingSince).getTime() : 0,
  );
  const slaDeadlineMs = $derived(
    app.slaDeadline ? new Date(app.slaDeadline).getTime() : 0,
  );
  // Prefer the client-side comparison while waiting so the badge transitions
  // live without needing a fresh API refresh; once finalised, trust the
  // workflow's stored value.
  const slaBreached = $derived(
    isWaiting && app.slaDeadline
      ? now > slaDeadlineMs
      : (app.slaBreached ?? false),
  );
  // Reference time for SLA duration. While waiting we use live `now` so the
  // figure ticks once a second; once the wait has resolved we freeze the
  // reference to `app.updatedAt` so the displayed duration is stable and does
  // not drift as wall-clock time keeps advancing.
  const slaReferenceTimeMs = $derived(
    isWaiting ? now : new Date(app.updatedAt).getTime(),
  );
  // Signed offset from the deadline. Negative while there is still time
  // remaining on the SLA, positive once the deadline has passed. The badge
  // already carries the outcome so this string only conveys timing context.
  const slaOffsetMs = $derived(slaReferenceTimeMs - slaDeadlineMs);
  const slaSignedDuration = $derived(
    slaOffsetMs > 0
      ? `+${formatDuration(slaOffsetMs)}`
      : `−${formatDuration(-slaOffsetMs)}`,
  );
  const slaProgress = $derived.by(() => {
    const total = slaDeadlineMs - pendingSinceMs;
    if (total <= 0) return slaBreached ? 1 : 0;
    return Math.min(1, Math.max(0, (now - pendingSinceMs) / total));
  });
  // Amber warning once more than half of the SLA window has elapsed but the
  // deadline has not yet been reached. Only meaningful while waiting; once
  // finalised the outcome is binary (met or breached).
  const slaWarning = $derived(isWaiting && !slaBreached && slaProgress > 0.5);
  // Stopped takes precedence over breach/warn/ok so a non-running terminal
  // workflow never displays a live business-outcome badge. Lifecycle states
  // are presented as their own distinct badge value.
  const slaState = $derived(
    stoppedStatus !== undefined
      ? 'stopped'
      : slaBreached
        ? 'breached'
        : slaWarning
          ? 'warn'
          : 'ok',
  );
  const slaBadgeLabel = $derived(
    stoppedStatus !== undefined
      ? lifecycleLabel(stoppedStatus)
      : slaBreached
        ? 'SLA breached'
        : slaWarning
          ? 'SLA at risk'
          : 'Within SLA',
  );
  // Show the SLA row whenever there is a deadline to display (in flight or
  // persisted), or when the workflow has been stopped while waiting on a
  // dependency that had an SLA recorded.
  const showSlaRow = $derived(
    !!app.slaDeadline &&
      (isWaiting || !!app.slaStatus || stoppedStatus !== undefined),
  );
  const dependencyLabel = $derived(
    app.pendingDependency ? humanise(app.pendingDependency) : '',
  );

  function humanise(value: string): string {
    return value.replace(/_/g, ' ');
  }

  function formatDuration(ms: number): string {
    const totalSeconds = Math.floor(ms / 1000);
    if (totalSeconds < 60) return `${totalSeconds}s`;
    const minutes = Math.floor(totalSeconds / 60);
    const seconds = totalSeconds % 60;
    return `${minutes}m ${seconds}s`;
  }

  function formatShortTime(iso: string): string {
    return new Date(iso).toLocaleTimeString(undefined, {
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
    });
  }
</script>

<section class="card">
  <div class="card-header">
    <h2>Application Summary</h2>
    <button class="btn-secondary" onclick={onRefresh} disabled={refreshing}>
      {refreshing ? 'Refreshing…' : 'Refresh'}
    </button>
  </div>
  {#if refreshError}
    <p class="error">{refreshError}</p>
  {/if}
  <dl class="summary">
    <dt>Application ID</dt>
    <dd class="mono">{app.applicationId}</dd>
    <dt>Applicant</dt>
    <dd>{app.applicantName}</dd>
    <dt>Status</dt>
    <dd>
      <span class="badge status-{app.status}">
        {statusLabel(app.status)}
      </span>
    </dd>
    <dt>Workflow Version</dt>
    <dd>
      <span
        class="badge"
        style={workflowVersionStyle(app.workflowVersion ?? 'unknown')}
      >
        {workflowVersionLabel(app.workflowVersion ?? 'unknown')}
      </span>
    </dd>
    {#if app.workerBuildId}
      <dt>Worker Build ID</dt>
      <dd class="mono">{app.workerBuildId}</dd>
    {/if}
    <dt>Current Step</dt>
    <dd class="mono">{app.currentStep}</dd>
    {#if app.offerId}
      <dt>Offer ID</dt>
      <dd class="mono">{app.offerId}</dd>
    {/if}
    {#if app.propertyValue !== undefined}
      <dt>Property Value</dt>
      <dd>
        {new Intl.NumberFormat('en-GB', {
          style: 'currency',
          currency: 'GBP',
          maximumFractionDigits: 0,
        }).format(app.propertyValue)}
      </dd>
    {/if}
    {#if isWaiting}
      <dt>Waiting For</dt>
      <dd>
        <span class="waiting-label">{dependencyLabel}</span>
      </dd>
    {/if}
    {#if showSlaRow && app.slaDeadline}
      <dt>SLA</dt>
      <dd>
        <span
          class="badge sla-badge"
          class:sla-ok={slaState === 'ok'}
          class:sla-warn={slaState === 'warn'}
          class:sla-breached={slaState === 'breached'}
          class:sla-stopped={slaState === 'stopped'}
        >
          {slaBadgeLabel}
        </span>
      </dd>
      <dt>Target</dt>
      <dd>
        <div class="mono">
          {formatShortTime(app.slaDeadline)} ({slaSignedDuration})
        </div>
        {#if isWaiting}
          <div
            class="sla-progress"
            class:sla-progress-warn={slaState === 'warn'}
            class:sla-progress-breached={slaState === 'breached'}
            role="progressbar"
            aria-valuemin="0"
            aria-valuemax="100"
            aria-valuenow={Math.round(slaProgress * 100)}
          >
            <div
              class="sla-progress-bar"
              style="width: {slaProgress * 100}%"
            ></div>
          </div>
        {/if}
      </dd>
    {/if}
    <dt>Created</dt>
    <dd>{formatTime(app.createdAt)}</dd>
    <dt>Updated</dt>
    <dd>{formatTime(app.updatedAt)}</dd>
  </dl>
  {#if !isTerminal}
    <p class="status-note">
      Auto-refreshing every {Math.round(refreshTimeout / 1000)}s.
    </p>
  {:else}
    <p class="status-note terminal">Workflow has reached a terminal state.</p>
  {/if}
</section>

<style>
  .card-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: var(--space-4);
    padding-bottom: var(--space-3);
    border-bottom: 1px solid var(--color-line-soft);
    gap: var(--space-3);
  }

  .card-header :global(h2) {
    margin-bottom: 0;
    padding-bottom: 0;
    border-bottom: none;
  }

  .summary {
    display: grid;
    grid-template-columns: auto 1fr;
    column-gap: var(--space-6);
    row-gap: 0;
    font-size: 14px;
  }

  .summary dt {
    color: var(--color-ink-muted);
    font-weight: 500;
    white-space: nowrap;
    padding: var(--space-2) 0;
    border-bottom: 1px solid var(--color-line-hair);
  }

  .summary dd {
    color: var(--color-ink);
    padding: var(--space-2) 0;
    word-break: break-all;
    border-bottom: 1px solid var(--color-line-hair);
    font-weight: 500;
  }

  .summary dt:last-of-type,
  .summary dd:last-of-type {
    border-bottom: none;
  }

  .status-note {
    font-size: 13px;
    color: var(--color-ink-muted);
    margin-top: var(--space-4);
    padding-top: var(--space-3);
    border-top: 1px solid var(--color-line-soft);
    display: flex;
    align-items: center;
    gap: var(--space-3);
  }

  .status-note::before {
    content: '';
    display: inline-block;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: var(--color-green-bright);
    box-shadow: 0 0 0 0 rgba(0, 184, 113, 0.45);
    animation: pulse 1.6s ease-out infinite;
  }

  .status-note.terminal {
    color: var(--color-green-darker);
  }

  .status-note.terminal::before {
    background: var(--color-green);
    animation: none;
  }

  @keyframes pulse {
    0% {
      box-shadow: 0 0 0 0 rgba(0, 184, 113, 0.45);
    }
    100% {
      box-shadow: 0 0 0 10px rgba(0, 184, 113, 0);
    }
  }

  /* Application status badge colours */
  .status-submitted {
    background: var(--color-slate-tint);
    color: var(--color-slate);
    border-color: var(--color-slate-line);
  }

  .status-credit_check_pending {
    background: var(--color-amber-tint);
    color: var(--color-amber);
    border-color: var(--color-amber-line);
  }

  .status-offer_reserved {
    background: var(--color-blue-tint);
    color: var(--color-blue);
    border-color: var(--color-blue-line);
  }

  .status-completed {
    background: var(--color-green-pale);
    color: var(--color-green-darker);
    border-color: var(--color-green);
  }

  .status-rejected {
    background: var(--color-red-tint);
    color: var(--color-red);
    border-color: var(--color-red-line);
  }

  .status-compensation_required {
    background: var(--color-amber-tint);
    color: var(--color-amber);
    border-color: var(--color-amber-line);
  }

  .status-compensated {
    background: var(--color-purple-tint);
    color: var(--color-purple);
    border-color: var(--color-purple-line);
  }

  /* SLA visibility for pending async dependencies */
  .waiting-label {
    margin-right: var(--space-2);
    text-transform: capitalize;
    color: var(--color-amber);
    font-weight: 600;
  }

  .sla-badge {
    font-size: 11px;
  }

  .sla-ok {
    background: var(--color-green-pale);
    color: var(--color-green-darker);
    border-color: var(--color-green);
  }

  .sla-warn {
    background: var(--color-amber-tint);
    color: var(--color-amber);
    border-color: var(--color-amber-line);
  }

  .sla-breached {
    background: var(--color-red-tint);
    color: var(--color-red);
    border-color: var(--color-red-line);
  }

  .sla-stopped {
    background: var(--color-slate-tint);
    color: var(--color-slate);
    border-color: var(--color-slate-line);
  }

  .sla-progress {
    margin-top: var(--space-2);
    width: 100%;
    height: 6px;
    background: var(--color-line-soft);
    border-radius: var(--radius-pill);
    overflow: hidden;
  }

  .sla-progress-bar {
    height: 100%;
    background: var(--color-green);
    border-radius: var(--radius-pill);
    transition:
      width 0.5s linear,
      background-color 0.2s linear;
  }

  .sla-progress-warn .sla-progress-bar {
    background-color: var(--color-amber);
  }

  .sla-progress-breached .sla-progress-bar {
    background-color: var(--color-red);
    width: 100% !important;
  }
</style>
