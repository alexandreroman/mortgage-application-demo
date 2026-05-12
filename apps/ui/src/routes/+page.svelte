<script lang="ts">
  import { goto } from '$app/navigation';
  import { resolve } from '$app/paths';
  import { page } from '$app/state';
  import { env } from '$env/dynamic/public';
  import * as api from '$lib/api';
  // Photo by Danielle Hoang on Unsplash. Free Unsplash licence.
  import heroImage from '$lib/assets/hero/streetscape.jpg';
  import ActionsPanel from '$lib/components/ActionsPanel.svelte';
  import ApplicationSummary from '$lib/components/ApplicationSummary.svelte';
  import AuditTimeline from '$lib/components/AuditTimeline.svelte';
  import { applicationExecutionKey } from '$lib/types';
  import type {
    ApplicationListItem,
    ApplicationWorkflowStatus,
    MortgageApplication,
    ScenarioOption,
  } from '$lib/types';
  import {
    workflowStatusLabel,
    workflowVersionLabel,
    workflowVersionStyle,
  } from '$lib/utils';
  import { untrack } from 'svelte';

  import type { PageData } from './$types';

  let { data }: { data: PageData } = $props();

  const bankName = env.PUBLIC_BANK_NAME ?? 'Temporal Bank';
  const refreshTimeout = 3000;

  // ── Scenarios ───────────────────────────────────────────────────────────
  const DEFAULT_SCENARIOS: ScenarioOption[] = [
    { name: 'happy_path', description: 'Full successful mortgage workflow.' },
    {
      name: 'fail_after_offer_reservation',
      description: 'Fails after the offer reservation step.',
    },
    {
      name: 'fail_and_compensate_after_offer_reservation',
      description: 'Fails and compensates after the offer reservation step.',
    },
  ];

  // untrack: intentional one-time snapshot from load data; polling/actions manage state locally.
  let scenarios: ScenarioOption[] = $state(untrack(() => data.scenarios));

  // ── Applications list ────────────────────────────────────────────────────
  let applications: ApplicationListItem[] = $state(
    untrack(() => data.applications),
  );

  function workflowStatusStyle(status: ApplicationWorkflowStatus): string {
    const map: Record<ApplicationWorkflowStatus, string> = {
      running:
        'background:var(--color-blue-tint);color:var(--color-blue);border-color:var(--color-blue-line)',
      completed:
        'background:var(--color-green-pale);color:var(--color-green-darker);border-color:var(--color-green)',
      failed:
        'background:var(--color-red-tint);color:var(--color-red);border-color:var(--color-red-line)',
      cancelled:
        'background:var(--color-slate-tint);color:var(--color-slate);border-color:var(--color-slate-line)',
      terminated:
        'background:var(--color-purple-tint);color:var(--color-purple);border-color:var(--color-purple-line)',
      timed_out:
        'background:var(--color-amber-tint);color:var(--color-amber);border-color:var(--color-amber-line)',
      continued_as_new:
        'background:var(--color-blue-tint);color:var(--color-blue);border-color:var(--color-blue-line)',
      unknown:
        'background:var(--color-slate-tint);color:var(--color-slate);border-color:var(--color-slate-line)',
    };
    return map[status];
  }

  const defaultName = 'Patrick Clifton';

  // ── Start form ──────────────────────────────────────────────────────────
  let startId = $state(crypto.randomUUID());
  let startName = $state(defaultName);
  let startScenario = $state('happy_path');
  let startFailureRate = $state(0);
  let startError = $state('');
  let startLoading = $state(false);

  // ── Load state ──────────────────────────────────────────────────────────
  let loadError = $state('');
  let loadLoading = $state(false);

  // ── Current application ─────────────────────────────────────────────────
  let app = $state<MortgageApplication | null>(untrack(() => data.app));
  let refreshing = $state(false);
  let refreshError = $state('');

  // ── Derived ─────────────────────────────────────────────────────────────
  const TERMINAL = new Set(['completed', 'rejected', 'compensated']);

  function allowsFailureInjection(scenario: string): boolean {
    return scenario === 'happy_path';
  }

  const scenarioOptions = $derived(
    scenarios.length > 0 ? scenarios : DEFAULT_SCENARIOS,
  );
  const selectedDesc = $derived(
    scenarioOptions.find((s) => s.name === startScenario)?.description ?? '',
  );
  const showFailureSlider = $derived(allowsFailureInjection(startScenario));
  // Action visibility is gated on the precise pending dependency reported by
  // the workflow rather than a broad business-status flag, so the form only
  // appears while the workflow is genuinely waiting for that specific
  // operator input.
  const isCreditCheckPending = $derived(
    app?.workflowStatus === 'running' &&
      app?.pendingDependency === 'credit_check',
  );
  // v2 workflow is waiting for the operator to submit a property value. This
  // is the gate that drives the "Submit Property Valuation" UI action and is
  // only ever true when the worker is the v2 build with no value submitted
  // yet.
  const isPropertyValuationPending = $derived(
    app?.workflowStatus === 'running' &&
      app?.workflowVersion === 'v2' &&
      app?.pendingDependency === 'property_valuation',
  );
  const isTerminal = $derived(app ? TERMINAL.has(app.status) : false);
  // Polling tracks the workflow's lifecycle state, NOT the business state.
  // The two diverge: the workflow can have business `status === 'completed'`
  // for several seconds while it is still running its tail activities (e.g.
  // SendNotification, with its random delay and retry policy). Stopping
  // polling on business completion would freeze the UI before the final
  // workflowStatus and audit entries arrive. Polling therefore continues
  // while `workflowStatus === 'running'` and only stops once a non-running
  // status has been observed and rendered. Undefined is treated as running
  // so an in-flight initial load never permanently freezes the UI.
  const isWorkflowRunning = $derived(
    !app?.workflowStatus || app.workflowStatus === 'running',
  );

  // Reset failure rate when switching away from happy_path so the slider
  // always starts at 0 when the user returns to happy_path.
  $effect(() => {
    if (!allowsFailureInjection(startScenario)) {
      startFailureRate = 0;
    }
  });

  // ── Sync from load data on navigation ───────────────────────────────────
  // The untrack snapshots above are initialised once. This effect re-syncs
  // them whenever SvelteKit re-runs load (e.g. navigating home clears app).
  $effect(() => {
    app = data.app ?? null;
    applications = data.applications;
    loadError = '';
  });

  // ── Polling via $effect ──────────────────────────────────────────────────
  $effect(() => {
    if (!app || !isWorkflowRunning) return;
    const timer = setInterval(() => void doRefresh(), refreshTimeout);
    return () => clearInterval(timer);
  });

  // ── Core actions ─────────────────────────────────────────────────────────
  async function refreshApplications() {
    try {
      applications = await api.getApplications();
    } catch {
      // fail softly — list will be stale but the create flow still succeeds
    }
  }

  async function doRefresh() {
    if (!app) return;
    refreshing = true;
    refreshError = '';
    try {
      app = await api.getApplication(app.applicationId, app.runId);
    } catch (e) {
      refreshError = e instanceof Error ? e.message : 'Refresh failed';
    } finally {
      refreshing = false;
    }
    await refreshApplications();
  }

  async function loadApplication(id: string, runId?: string) {
    loadLoading = true;
    loadError = '';
    try {
      app = await api.getApplication(id, runId);

      const url = new URL(resolve('/'), page.url.origin);
      url.searchParams.set('applicationId', id);
      if (runId) {
        url.searchParams.set('runId', runId);
      } else {
        url.searchParams.delete('runId');
      }

      // eslint-disable-next-line svelte/no-navigation-without-resolve
      await goto(url, {
        replaceState: true,
        keepFocus: true,
        noScroll: true,
      });
    } catch (e) {
      loadError = e instanceof Error ? e.message : 'Failed to load application';
      app = null;
    } finally {
      loadLoading = false;
    }
  }

  async function clearSelection() {
    const url = new URL(resolve('/'), page.url.origin);
    // eslint-disable-next-line svelte/no-navigation-without-resolve
    await goto(url, {
      replaceState: true,
      keepFocus: true,
      noScroll: true,
    });
  }

  // ── Form handlers ─────────────────────────────────────────────────────────
  async function handleRerun(newApplicationId: string) {
    await loadApplication(newApplicationId);
    await refreshApplications();
  }

  async function handleStart(e: SubmitEvent) {
    e.preventDefault();
    startLoading = true;
    startError = '';
    try {
      await api.startApplication({
        applicationId: startId,
        applicantName: startName.trim(),
        scenario: startScenario,
        externalFailureRatePercent: allowsFailureInjection(startScenario)
          ? startFailureRate
          : 0,
      });
      const launched = startId;
      startId = crypto.randomUUID();
      startName = defaultName;
      await loadApplication(launched);
      await refreshApplications();
    } catch (e) {
      startError =
        e instanceof Error ? e.message : 'Failed to start application';
    } finally {
      startLoading = false;
    }
  }
</script>

<svelte:head>
  <title>Mortgage Application Console</title>
</svelte:head>

<header class="topbar">
  <div class="topbar-inner">
    <span class="topbar-eyebrow">Mortgage demo · Operator console</span>
  </div>
</header>

<header class="brand-bar">
  <div class="brand-bar-inner">
    <a href={resolve('/')} class="brand-lockup" aria-label={bankName}>
      <svg
        class="brand-mark"
        viewBox="0 0 45 45"
        xmlns="http://www.w3.org/2000/svg"
        aria-hidden="true"
      >
        <path
          fill="#0a1d14"
          d="M22 9c10.5 1 16.5 4.5 16.5 8.75 0 4.5-5 7-9 9-5 2.5-6.5 5-6.5 9 0 2 5 9 5 14h-15v-2c0-1 1-2 1-3 0-1.5-0.5-2.5-1.5-3.5-2.5-2.5-7.5-2.5-10-5-2-2-2-7-1-10 1-2.5 3-3.5 3-5.5 0-3-1.5-5-3-5 0 0-1-5 4-6z"
        />
      </svg>
      <span class="brand-wordmark">{bankName.toUpperCase()}</span>
    </a>
    <nav class="brand-utility" aria-label="Operator utility">
      <a
        class="utility-link"
        href="http://localhost:8233"
        target="_blank"
        rel="noopener noreferrer"
      >
        Temporal UI
      </a>
      <a
        class="utility-link"
        href="http://localhost:3000"
        target="_blank"
        rel="noopener noreferrer"
      >
        Grafana
      </a>
    </nav>
  </div>
</header>

<section
  class="hero"
  aria-labelledby="hero-title"
  style="background-image: url({heroImage});"
>
  <div class="hero-scrim" aria-hidden="true"></div>
  <div class="hero-inner">
    <div class="hero-card">
      <h1 id="hero-title">Bring every mortgage home, on time.</h1>
      <p class="hero-sub">
        Start, track, and resolve mortgage applications running on Temporal. A
        durable view of every step, every wait, every retry.
      </p>
    </div>
  </div>
</section>

<div class="console">
  <div class="workspace">
    <!-- ── Sidebar rail ─────────────────────────────────────────────── -->
    <aside class="rail" aria-label="Applications">
      <div class="rail-section">
        <div class="rail-header">
          <p class="rail-title">Applications</p>
          {#if app}
            <button
              type="button"
              class="btn-primary rail-newbtn"
              onclick={() => void clearSelection()}
            >
              + New
            </button>
          {/if}
        </div>
        {#if applications.length === 0}
          <p class="muted rail-empty">No applications yet.</p>
        {:else}
          <ul class="app-list">
            {#each applications as item (applicationExecutionKey(item))}
              <li>
                <button
                  type="button"
                  class="app-tile"
                  class:app-tile--active={app?.applicationId ===
                    item.applicationId && app?.runId === item.runId}
                  onclick={() =>
                    void loadApplication(item.applicationId, item.runId)}
                  disabled={loadLoading}
                >
                  <div class="app-tile-info">
                    <span class="app-tile-name">{item.applicantName}</span>
                    <span class="mono app-tile-id">{item.applicationId}</span>
                  </div>
                  <div class="app-tile-meta">
                    <span
                      class="badge"
                      style={workflowVersionStyle(
                        item.workflowVersion ?? 'unknown',
                      )}
                      title={item.workerBuildId
                        ? `Worker Build ID: ${item.workerBuildId}`
                        : 'Workflow version'}
                    >
                      {workflowVersionLabel(item.workflowVersion ?? 'unknown')}
                    </span>
                    <span
                      class="badge"
                      style={workflowStatusStyle(item.workflowStatus)}
                    >
                      {workflowStatusLabel(item.workflowStatus)}
                    </span>
                  </div>
                </button>
              </li>
            {/each}
          </ul>
        {/if}
        {#if loadError}
          <p class="error rail-error">{loadError}</p>
        {/if}
      </div>
    </aside>

    <!-- ── Main panel ───────────────────────────────────────────────── -->
    <main class="workspace-main">
      {#if app}
        <ApplicationSummary
          {app}
          {isTerminal}
          {refreshing}
          {refreshError}
          onRefresh={doRefresh}
          {refreshTimeout}
        />
        <ActionsPanel
          {app}
          {isTerminal}
          {isCreditCheckPending}
          {isPropertyValuationPending}
          onRefresh={doRefresh}
          onRerun={handleRerun}
        />
        <AuditTimeline timeline={app.timeline} />
      {:else}
        <section class="card">
          <h2>Start application</h2>
          <form onsubmit={handleStart} class="start-form">
            <!-- Bank application fields -->
            <div class="form-bank">
              <div class="field">
                <label for="start-id">Application ID</label>
                <div class="input-row">
                  <input
                    id="start-id"
                    type="text"
                    bind:value={startId}
                    placeholder="UUID"
                    required
                    class="mono"
                  />
                  <button
                    type="button"
                    class="btn-secondary"
                    onclick={() => (startId = crypto.randomUUID())}>New</button
                  >
                </div>
              </div>

              <div class="field">
                <label for="start-name">Applicant Name</label>
                <input
                  id="start-name"
                  type="text"
                  bind:value={startName}
                  required
                />
              </div>
            </div>

            <!-- Temporal demo controls -->
            <div class="demo-controls">
              <p class="demo-controls-title">Temporal Demo Controls</p>
              <p class="demo-controls-hint">
                These controls simulate demo conditions such as workflow
                scenarios and unreliable external dependencies. They are not
                bank operator controls.
              </p>
              <div class="demo-fields">
                <div class="field">
                  <label for="start-scenario">Scenario</label>
                  <select id="start-scenario" bind:value={startScenario}>
                    {#each scenarioOptions as s (s.name)}
                      <option value={s.name}>{s.name}</option>
                    {/each}
                  </select>
                  {#if selectedDesc}
                    <p class="hint">{selectedDesc}</p>
                  {/if}
                </div>

                {#if showFailureSlider}
                  <div class="field">
                    <label for="start-failure-rate"
                      >External failure rate <span class="rate-value"
                        >{startFailureRate}%</span
                      ></label
                    >
                    <input
                      id="start-failure-rate"
                      type="range"
                      min="0"
                      max="75"
                      step="5"
                      bind:value={startFailureRate}
                    />
                    <p class="hint">
                      Simulates flaky external systems by randomly failing
                      eligible activities. Temporal retries should absorb
                      transient failures.
                    </p>
                  </div>
                {/if}
              </div>
            </div>

            {#if startError}
              <p class="error">{startError}</p>
            {/if}
            <div class="form-actions">
              <button type="submit" class="btn-primary" disabled={startLoading}>
                {startLoading ? 'Starting…' : 'Start Workflow'}
              </button>
            </div>
          </form>
        </section>
      {/if}
    </main>
  </div>
</div>

<style>
  /* ── Top eyebrow bar (Lloyds audience nav band) ──────────────────────── */
  .topbar {
    background: #000000;
    color: #ffffff;
  }

  .topbar-inner {
    max-width: 1280px;
    margin: 0 auto;
    padding: var(--space-2) var(--space-6);
    display: flex;
    align-items: center;
  }

  .topbar-eyebrow {
    font-size: 13px;
    font-weight: 600;
    letter-spacing: 0.01em;
    color: rgba(255, 255, 255, 0.92);
  }

  /* ── Brand bar ──────────────────────────────────────────────────────────── */
  .brand-bar {
    background: var(--color-green);
    color: #ffffff;
  }

  .brand-bar-inner {
    max-width: 1280px;
    margin: 0 auto;
    padding: var(--space-4) var(--space-6);
    display: flex;
    align-items: center;
    gap: var(--space-6);
  }

  .brand-lockup {
    display: inline-flex;
    align-items: center;
    gap: var(--space-3);
    text-decoration: none;
    color: inherit;
  }

  .brand-mark {
    width: 34px;
    height: 34px;
    background: #ffffff;
    border-radius: var(--radius-pill);
    padding: var(--space-1);
  }

  .brand-wordmark {
    font-size: 24px;
    font-weight: 800;
    letter-spacing: 0.04em;
    color: #ffffff;
  }

  .brand-utility {
    margin-left: auto;
    display: flex;
    align-items: center;
    gap: var(--space-6);
  }

  .utility-link {
    color: #ffffff;
    text-decoration: none;
    font-size: 14px;
    font-weight: 600;
    letter-spacing: 0.01em;
    padding: var(--space-1) 0;
    border-bottom: 2px solid transparent;
    transition: border-color 140ms ease;
    display: inline-flex;
    align-items: center;
    gap: var(--space-2);
  }

  .utility-link::after {
    content: '↗';
    font-size: 12px;
    opacity: 0.85;
  }

  .utility-link:hover {
    border-color: #ffffff;
  }

  @media (max-width: 640px) {
    .brand-utility {
      display: none;
    }
  }

  /* ── Hero banner (full-bleed photo + overlay card) ──────────────────────── */
  .hero {
    position: relative;
    background-color: var(--color-ink);
    background-size: cover;
    background-position: center 60%;
    background-repeat: no-repeat;
    isolation: isolate;
  }

  .hero-scrim {
    position: absolute;
    inset: 0;
    background: linear-gradient(
      90deg,
      rgba(10, 29, 20, 0.55) 0%,
      rgba(10, 29, 20, 0.2) 55%,
      rgba(10, 29, 20, 0) 100%
    );
    z-index: 0;
  }

  .hero-inner {
    position: relative;
    z-index: 1;
    max-width: 1280px;
    margin: 0 auto;
    padding: clamp(var(--space-6), 4vw, 56px) var(--space-6);
    min-height: 380px;
    display: flex;
    align-items: center;
  }

  .hero-card {
    background: var(--color-green-darker);
    color: #ffffff;
    padding: clamp(var(--space-6), 3vw, 40px) clamp(var(--space-6), 3vw, 40px);
    border-radius: var(--radius-lg);
    width: min(100%, 520px);
    box-shadow: 0 30px 60px -24px rgba(10, 29, 20, 0.55);
  }

  .hero-card h1 {
    font-family: var(--font-display);
    font-size: clamp(36px, 4.6vw, 56px);
    font-weight: 700;
    letter-spacing: -0.012em;
    line-height: 1.02;
    color: #ffffff;
  }

  .hero-sub {
    margin-top: var(--space-5);
    font-size: 16px;
    line-height: 1.55;
    color: rgba(255, 255, 255, 0.92);
  }

  @media (max-width: 720px) {
    .hero-inner {
      min-height: 280px;
    }
    .hero-card h1 {
      font-size: 32px;
    }
  }

  /* ── Page layout ────────────────────────────────────────────────────────── */
  .console {
    max-width: 1280px;
    margin: 0 auto;
    padding: var(--space-7) var(--space-6) var(--space-8);
    display: flex;
    flex-direction: column;
    gap: var(--space-6);
  }

  /* ── Workspace (rail + main) ────────────────────────────────────────────── */
  .workspace {
    display: grid;
    grid-template-columns: 320px 1fr;
    gap: var(--space-6);
    align-items: start;
  }

  @media (max-width: 900px) {
    .workspace {
      grid-template-columns: 1fr;
    }
  }

  .rail {
    position: sticky;
    top: var(--space-6);
    display: flex;
    flex-direction: column;
    gap: var(--space-4);
  }

  @media (max-width: 900px) {
    .rail {
      position: static;
    }
  }

  .rail-section {
    background: var(--color-surface);
    border: 1px solid var(--color-line);
    border-radius: var(--radius-lg);
    padding: var(--space-5);
    box-shadow: var(--shadow-card);
  }

  .rail-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: var(--space-3);
    margin-bottom: var(--space-3);
  }

  .rail-title {
    font-size: 12px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    color: var(--color-ink-muted);
    margin: 0;
  }

  .rail-newbtn {
    padding: var(--space-2) var(--space-4);
    font-size: 13px;
  }

  .rail-empty {
    font-size: 13px;
    color: var(--color-ink-muted);
  }

  .rail-error {
    margin-top: var(--space-3);
  }

  .workspace-main {
    display: flex;
    flex-direction: column;
    gap: var(--space-5);
    min-width: 0;
  }

  /* ── Start form layout ──────────────────────────────────────────────────── */
  .start-form {
    display: flex;
    flex-direction: column;
    gap: 0;
  }

  .form-bank {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0 var(--space-6);
    align-items: start;
    padding-bottom: var(--space-5);
  }

  @media (max-width: 700px) {
    .form-bank {
      grid-template-columns: 1fr;
    }
  }

  /* ── Demo controls section ──────────────────────────────────────────────── */
  .demo-controls {
    padding: var(--space-5);
    background: var(--color-surface-alt);
    border: 1px solid var(--color-line-soft);
    border-radius: var(--radius-md);
  }

  .demo-controls-title {
    font-size: 15px;
    font-weight: 700;
    color: var(--color-ink);
    margin: 0 0 var(--space-2);
  }

  .demo-controls-hint {
    font-size: 13px;
    color: var(--color-ink-muted);
    margin: 0 0 var(--space-4);
    line-height: 1.5;
  }

  .demo-fields {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0 var(--space-6);
    align-items: start;
  }

  @media (max-width: 700px) {
    .demo-fields {
      grid-template-columns: 1fr;
    }
  }

  .rate-value {
    font-weight: 600;
    color: var(--color-green-dark);
    font-variant-numeric: tabular-nums;
    margin-left: var(--space-2);
  }

  input[type='range'] {
    width: 100%;
    accent-color: var(--color-green);
    margin-top: var(--space-2);
  }

  .input-row {
    display: flex;
    gap: var(--space-2);
  }

  .input-row input {
    flex: 1;
    min-width: 0;
  }

  /* ── Applications list (Lloyds-style tiles in left rail) ──────────────── */
  .app-list {
    list-style: none;
    padding: 0;
    margin: 0;
    display: flex;
    flex-direction: column;
    gap: var(--space-2);
  }

  .app-tile {
    position: relative;
    display: flex;
    flex-direction: column;
    gap: var(--space-2);
    width: 100%;
    padding: var(--space-3) var(--space-7) var(--space-3) var(--space-4);
    border: 1.5px solid var(--color-line);
    border-radius: var(--radius-md);
    background: var(--color-surface);
    cursor: pointer;
    text-align: left;
    transition:
      background 140ms ease,
      border-color 140ms ease;
    font-family: var(--font-body);
  }

  .app-tile::after {
    content: '›';
    position: absolute;
    right: var(--space-3);
    top: 50%;
    transform: translateY(-55%);
    color: var(--color-green);
    font-size: 24px;
    font-weight: 400;
    line-height: 1;
  }

  .app-tile:hover:not(:disabled) {
    background: var(--color-green-tint);
    border-color: var(--color-green);
  }

  .app-tile--active {
    background: var(--color-green-pale);
    border-color: var(--color-green);
  }

  .app-tile--active .app-tile-name {
    color: var(--color-green-darker);
  }

  .app-tile:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .app-tile-info {
    display: flex;
    flex-direction: column;
    gap: var(--space-1);
    min-width: 0;
  }

  .app-tile-name {
    font-size: 15px;
    font-weight: 700;
    color: var(--color-ink);
    letter-spacing: -0.005em;
  }

  .app-tile-id {
    font-size: 11px;
    color: var(--color-ink-muted);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .app-tile-meta {
    display: flex;
    gap: var(--space-2);
    align-items: center;
    flex-wrap: wrap;
  }
</style>
