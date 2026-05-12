<script lang="ts">
  import * as api from '$lib/api';
  import type { MortgageApplication } from '$lib/types';
  import { statusLabel } from '$lib/utils';

  let {
    app,
    isTerminal,
    isCreditCheckPending,
    isPropertyValuationPending,
    onRefresh,
    onRerun,
  }: {
    app: MortgageApplication;
    isTerminal: boolean;
    isCreditCheckPending: boolean;
    isPropertyValuationPending: boolean;
    onRefresh: () => Promise<void>;
    onRerun: (applicationId: string) => Promise<void>;
  } = $props();

  let creditResult: 'approved' | 'rejected' = $state('approved');
  let creditRef = $state('');
  let creditError = $state('');
  let creditLoading = $state(false);

  let retryError = $state('');
  let retryLoading = $state(false);

  let rerunError = $state('');
  let rerunLoading = $state(false);

  // Default property value for the v2 valuation form. The API does NOT
  // apply a default — this is purely a demo convenience so the operator can
  // submit a sensible value with a single click. £350,000 is the canonical
  // demo figure.
  const PROPERTY_VALUE_DEFAULT = 350000;
  let propertyValue = $state<number>(PROPERTY_VALUE_DEFAULT);
  let propertyValuationError = $state('');
  let propertyValuationLoading = $state(false);

  // Tracks the most recent manual action the operator has submitted, scoped
  // to the application it was sent for. While set, the corresponding form is
  // hidden so the same form does not briefly reappear between submission
  // and the next backend refresh catching up to the new workflow state.
  // The pairing with applicationId guards against stale state if the
  // operator switches to a different application before the refresh lands.
  let recentlySubmittedAction = $state<
    | {
        applicationId: string;
        type: 'submit_credit_check_result' | 'submit_property_valuation';
      }
    | undefined
  >();

  // Clear the just-submitted flag once fresh backend state proves the
  // workflow has moved past the wait we unblocked: the workflow is no
  // longer running, the operator has navigated to a different application,
  // or the pending dependency has changed.
  $effect(() => {
    if (!recentlySubmittedAction) return;
    if (recentlySubmittedAction.applicationId !== app.applicationId) {
      recentlySubmittedAction = undefined;
      return;
    }
    if (app.workflowStatus && app.workflowStatus !== 'running') {
      recentlySubmittedAction = undefined;
      return;
    }
    if (
      recentlySubmittedAction.type === 'submit_credit_check_result' &&
      app.pendingDependency !== 'credit_check'
    ) {
      recentlySubmittedAction = undefined;
      return;
    }
    if (
      recentlySubmittedAction.type === 'submit_property_valuation' &&
      app.pendingDependency !== 'property_valuation'
    ) {
      recentlySubmittedAction = undefined;
    }
  });

  // Effective form visibility. The parent decides whether the workflow is
  // waiting for a given input; we additionally suppress the form once the
  // operator has just submitted it, so the form disappears immediately on
  // success rather than lingering until polling catches up.
  const showCreditCheckForm = $derived(
    isCreditCheckPending &&
      recentlySubmittedAction?.type !== 'submit_credit_check_result',
  );
  const showPropertyValuationForm = $derived(
    isPropertyValuationPending &&
      recentlySubmittedAction?.type !== 'submit_property_valuation',
  );

  // Passive "workflow is continuing" cue. Shown when the workflow is still
  // running but no operator form is visible — either because nothing is
  // waiting for input right now, or because the operator has just submitted
  // an action and the UI is waiting for backend state to catch up.
  // workflowStatus is treated as running when undefined to match the rest of
  // the UI's lifecycle handling for older payloads.
  const showContinuingCue = $derived(
    !isTerminal &&
      !showCreditCheckForm &&
      !showPropertyValuationForm &&
      (!app.workflowStatus || app.workflowStatus === 'running'),
  );

  // Format an integer pound value as £350,000 for display next to the input.
  // Rendered through Intl rather than a hand-rolled regex so it follows the
  // user's locale where applicable.
  function formatGbp(value: number): string {
    if (!Number.isFinite(value)) return '';
    return new Intl.NumberFormat('en-GB', {
      style: 'currency',
      currency: 'GBP',
      maximumFractionDigits: 0,
    }).format(value);
  }

  async function handleCreditCheck(e: SubmitEvent) {
    e.preventDefault();
    creditLoading = true;
    creditError = '';
    try {
      await api.performAction(app.applicationId, {
        type: 'submit_credit_check_result',
        payload: {
          result: creditResult,
          reference: creditRef.trim() || undefined,
        },
      });
      creditRef = '';
      recentlySubmittedAction = {
        applicationId: app.applicationId,
        type: 'submit_credit_check_result',
      };
      await onRefresh();
    } catch (err) {
      creditError =
        err instanceof Error ? err.message : 'Failed to submit credit check';
    } finally {
      creditLoading = false;
    }
  }

  async function handlePropertyValuation(e: SubmitEvent) {
    e.preventDefault();
    propertyValuationLoading = true;
    propertyValuationError = '';
    try {
      // Coerce defensively: a number-typed bound input can still produce
      // NaN if the field is cleared. Guard at the boundary so a malformed
      // value is caught before it hits the API.
      const value = Number(propertyValue);
      if (!Number.isFinite(value) || value <= 0) {
        propertyValuationError = 'Property value must be a positive number';
        return;
      }
      await api.performAction(app.applicationId, {
        type: 'submit_property_valuation',
        propertyValuation: { propertyValue: value },
      });
      recentlySubmittedAction = {
        applicationId: app.applicationId,
        type: 'submit_property_valuation',
      };
      await onRefresh();
    } catch (err) {
      propertyValuationError =
        err instanceof Error
          ? err.message
          : 'Failed to submit property valuation';
    } finally {
      propertyValuationLoading = false;
    }
  }

  async function handleRetryCreditCheck() {
    retryLoading = true;
    retryError = '';
    try {
      await api.performAction(app.applicationId, {
        type: 'retry_credit_check',
      });
      await onRefresh();
    } catch (err) {
      retryError =
        err instanceof Error ? err.message : 'Failed to retry credit check';
    } finally {
      retryLoading = false;
    }
  }

  async function handleRerun() {
    rerunLoading = true;
    rerunError = '';
    try {
      const result = await api.performAction(app.applicationId, {
        type: 'rerun_application',
      });
      if (result) {
        await onRerun(result.applicationId);
      }
    } catch (err) {
      rerunError =
        err instanceof Error ? err.message : 'Failed to re-run application';
    } finally {
      rerunLoading = false;
    }
  }
</script>

<section class="card">
  <h2>Available Actions</h2>
  {#if isTerminal}
    <p class="muted">
      No actions available. The workflow is in a terminal state:
      <strong>{statusLabel(app.status)}</strong>.
    </p>
  {:else if showPropertyValuationForm}
    <div class="action-block">
      <h3>Submit Property Valuation</h3>
      <p class="hint">
        The v2 workflow is waiting for the property valuation. Submit a value in
        pounds to unblock the offer reservation step.
      </p>
      <form onsubmit={handlePropertyValuation}>
        <div class="field">
          <label for="property-value">
            Property value <span class="rate-value"
              >{formatGbp(propertyValue)}</span
            >
          </label>
          <input
            id="property-value"
            type="number"
            min="1"
            step="1"
            bind:value={propertyValue}
            required
          />
        </div>
        {#if propertyValuationError}
          <p class="error">{propertyValuationError}</p>
        {/if}
        <div class="form-actions">
          <button
            type="submit"
            class="btn-primary"
            disabled={propertyValuationLoading}
          >
            {propertyValuationLoading
              ? 'Submitting…'
              : 'Submit Property Valuation'}
          </button>
        </div>
      </form>
    </div>
  {:else if showCreditCheckForm}
    <div class="action-block">
      <h3>Submit Credit Check Result</h3>
      <p class="hint">
        The workflow is waiting for an external credit check signal. Submit the
        outcome below to unblock it.
      </p>
      <form onsubmit={handleCreditCheck}>
        <div class="field">
          <label for="credit-result">Result</label>
          <select id="credit-result" bind:value={creditResult}>
            <option value="approved">Approved</option>
            <option value="rejected">Rejected</option>
          </select>
        </div>
        <div class="field">
          <label for="credit-ref">Reference (optional)</label>
          <input
            id="credit-ref"
            type="text"
            bind:value={creditRef}
            placeholder="e.g. CB-12345"
            class="mono"
          />
        </div>
        {#if creditError}
          <p class="error">{creditError}</p>
        {/if}
        <div class="form-actions">
          <button type="submit" class="btn-primary" disabled={creditLoading}>
            {creditLoading ? 'Submitting…' : 'Submit Credit Check'}
          </button>
        </div>
      </form>
    </div>
  {:else if showContinuingCue}
    <div class="continuing-cue" role="status" aria-live="polite">
      <h3>Processing&hellip;</h3>
      <p>
        The workflow is continuing and the next steps are being executed. This
        may take a few seconds.
      </p>
    </div>
  {/if}

  <div class="operator-section">
    <h3>Operator Controls</h3>
    <div class="operator-actions">
      <div class="operator-action">
        <button
          type="button"
          class="btn-secondary"
          onclick={handleRetryCreditCheck}
          disabled={retryLoading || isTerminal}
        >
          {retryLoading ? 'Retrying…' : 'Retry Credit Check'}
        </button>
        <p class="hint">Re-request the credit check for this application.</p>
        {#if retryError}
          <p class="error">{retryError}</p>
        {/if}
      </div>
      <div class="operator-action">
        <button
          type="button"
          class="btn-secondary"
          onclick={handleRerun}
          disabled={rerunLoading}
        >
          {rerunLoading ? 'Re-running…' : 'Re-run Application'}
        </button>
        <p class="hint">Start a new workflow with the same inputs.</p>
        {#if rerunError}
          <p class="error">{rerunError}</p>
        {/if}
      </div>
    </div>
  </div>
</section>

<style>
  .action-block {
    border: 1px solid var(--color-amber-line);
    background: var(--color-amber-tint);
    border-radius: var(--radius-md);
    padding: var(--space-4) var(--space-5);
  }

  .action-block h3 {
    font-size: 16px;
    font-weight: 700;
    color: var(--color-ink);
    margin-bottom: var(--space-2);
  }

  .action-block :global(.hint) {
    margin-bottom: var(--space-3);
    color: var(--color-ink-soft);
  }

  /* Passive cue shown while Temporal is progressing the workflow. */
  .continuing-cue {
    border: 1px solid var(--color-green);
    background: var(--color-green-pale);
    border-radius: var(--radius-md);
    padding: var(--space-4) var(--space-5);
  }

  .continuing-cue h3 {
    font-size: 16px;
    font-weight: 700;
    color: var(--color-green-darker);
    margin-bottom: var(--space-2);
    display: flex;
    align-items: center;
    gap: var(--space-3);
  }

  .continuing-cue h3::before {
    content: '';
    display: inline-block;
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background: var(--color-green-bright);
    box-shadow: 0 0 0 0 rgba(0, 184, 113, 0.5);
    animation: ring 1.6s ease-out infinite;
  }

  @keyframes ring {
    0% {
      box-shadow: 0 0 0 0 rgba(0, 184, 113, 0.55);
    }
    100% {
      box-shadow: 0 0 0 12px rgba(0, 184, 113, 0);
    }
  }

  .continuing-cue p {
    font-size: 13px;
    color: var(--color-green-darker);
    margin: 0;
    line-height: 1.5;
  }

  .rate-value {
    font-weight: 600;
    color: var(--color-green-dark);
    margin-left: var(--space-2);
    font-variant-numeric: tabular-nums;
  }

  .operator-section {
    margin-top: var(--space-5);
    padding-top: var(--space-4);
    border-top: 1px solid var(--color-line-soft);
  }

  .operator-section h3 {
    font-size: 14px;
    font-weight: 700;
    color: var(--color-ink);
    margin-bottom: var(--space-3);
  }

  .operator-actions {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
    gap: var(--space-3) var(--space-5);
  }

  .operator-action {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: var(--space-2);
  }

  .operator-action :global(.hint) {
    font-size: 12.5px;
    color: var(--color-ink-muted);
    margin: 0;
  }
</style>
