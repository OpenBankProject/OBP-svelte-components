# Sync Package from Portal

The OBP-Portal (`../OBP-Portal`) has evolved beyond this package in several areas. This document lists every change that needs to be applied to the package to bring it up to date. Apply all changes to the package source at `src/lib/`.

**Important context:** This package uses dependency injection (constructor params, factory functions) instead of singletons and `$env` imports. When porting code from the portal, replace direct singleton/env usage with injected dependencies.

---

## 1. Opey: Add `auth_refresh_needed` event and 401 handling

The portal added a complete auth-refresh flow for expired tokens. The package is missing this entirely.

### 1a. `src/lib/opey/services/ChatService.ts` — Add event types

Add `timestamp` to the `consent_request` event, make `bankId` optional, and add the `auth_refresh_needed` event to the `StreamEvent` union.

Replace the `consent_request` variant (currently lines 68-76):

```typescript
    | {
        type: 'consent_request',
        toolCallId: string,
        toolName: string,
        operationId: string | null,
        requiredRoles: string[],
        toolCallCount: number,
        bankId: string | null
      }
```

With:

```typescript
    | {
        type: 'consent_request',
        toolCallId: string,
        toolName: string,
        operationId: string | null,
        requiredRoles: string[],
        timestamp: number,
        toolCallCount: number,
        bankId?: string | null
      }
```

Then add this new variant after the `error` event (after line 78):

```typescript
    | { type: 'auth_refresh_needed' }
```

### 1b. `src/lib/opey/services/RestChatService.ts` — Add 401 detection

In the `handleStreamingResponse` method, after the `if (!res.ok) {` check, add 401 handling before the generic error path:

```typescript
            if (!res.ok) {
                // Handle 401 - token expired, need to refresh session
                if (res.status === 401) {
                    logger.info('Received 401 from Opey - token may have expired, requesting auth refresh');
                    this.streamEventCallback?.({ type: 'auth_refresh_needed' });
                    return;
                }

                let errorMessage = ...  // existing error handling continues
```

Also in `sendConsentResponse`, update logging to include `hasJwt` and the payload:

```typescript
    async sendConsentResponse(toolCallId: string, consentJwt: string | null, threadId: string): Promise<void> {
        logger.info(`Sending consent response for toolCallId=${toolCallId}, threadId=${threadId}, hasJwt=${!!consentJwt}`);

        const payload = {
            message: "",
            thread_id: threadId,
            tool_call_approval: consentJwt !== null
                ? { consent_jwt: consentJwt }
                : { consent_denied: true }
        };

        logger.info(`Consent payload:`, JSON.stringify(payload, null, 2));
        ...
```

Also in `handleStreamEvent`, for the `consent_request` case, add `timestamp` and use `??` for `toolCallCount`:

```typescript
            case 'consent_request':
                this.streamEventCallback?.({
                    type: 'consent_request',
                    toolCallId: eventData.tool_call_id,
                    toolName: eventData.tool_name,
                    operationId: eventData.operation_id || null,
                    requiredRoles: eventData.required_roles || [],
                    timestamp: eventData.timestamp || Date.now() / 1000,
                    toolCallCount: eventData.tool_call_count ?? 1,
                    bankId: eventData.bank_id || null
                });
                break;
```

### 1c. `src/lib/opey/controllers/ChatController.ts` — Add auth refresh callback

Add a private field:

```typescript
export class ChatController {
    private toolInstanceCounts: Record<string, number> = {};
    private authRefreshCallback?: () => Promise<void>;
```

Add a new case at the top of the `switch (event.type)` block in the constructor:

```typescript
            switch (event.type) {
                case 'auth_refresh_needed':
                    logger.info('Auth refresh needed - triggering callback');
                    if (this.authRefreshCallback) {
                        this.authRefreshCallback();
                    }
                    break;
                case 'user_message_confirmed':
                    ...
```

Add this public method at the end of the class:

```typescript
    /**
     * Register a callback to be called when auth refresh is needed (401 from Opey).
     * The callback should refresh the session and return a promise that resolves when done.
     */
    onAuthRefreshNeeded(callback: () => Promise<void>): void {
        this.authRefreshCallback = callback;
    }
```

### 1d. `src/lib/opey/state/ChatState.ts` — Fix consent field handling

In `addConsentRequest`, change the `consentBankId` assignment from always-assign to conditional:

```typescript
        // Change from:
        toolMessage.consentBankId = bankId;
        // To:
        if (bankId) toolMessage.consentBankId = bankId;
```

And change `consentOperationId` from `??` to `||`:

```typescript
        // Change from:
        toolMessage.consentOperationId = operationId ?? undefined;
        // To:
        toolMessage.consentOperationId = operationId || undefined;
```

---

## 2. Components: Update 3 tool-message components

The portal has significantly improved the UI for tool response components with proper card layouts, Lucide icons, copy buttons, and collapsible details.

### 2a. `src/lib/components/tool-messages/DefaultToolResponse.svelte`

Replace the entire file with:

```svelte
<script lang="ts">
    import type { ToolMessage } from "$lib/opey/types";
    import { toast } from '$lib/utils/toastService';
    import { Copy, CheckCircle } from '@lucide/svelte';

    let { message }: { message: ToolMessage } = $props();

    let outputContent = $derived(
        message.toolOutput
            ? typeof message.toolOutput === "string"
                ? message.toolOutput
                : JSON.stringify(message.toolOutput, null, 2)
            : "No output available."
    );

    let showOutput = $state(false);

    async function copyToClipboard() {
        try {
            await navigator.clipboard.writeText(outputContent);
            toast.info('Output copied to clipboard!');
        } catch (err) {
            console.error('Failed to copy: ', err);
            toast.error('Failed to copy to clipboard.');
        }
    }
</script>

<div class="card rounded-lg border border-surface-300-700 bg-surface-50-950 p-4 text-left">
    <!-- Header -->
    <div class="mb-3 flex items-center justify-between">
        <div class="flex items-center gap-2">
            <CheckCircle class="text-success-600-400" size={20} />
            <h4 class="text-sm font-semibold">Tool Output</h4>
        </div>
        <button
            type="button"
            class="btn btn-sm preset-tonal-primary"
            onclick={copyToClipboard}
            title="Copy output"
            aria-label="Copy output"
        >
            <Copy size={16} />
            <span class="hidden sm:inline">Copy</span>
        </button>
    </div>

    <!-- Tool Name -->
    <div class="mb-3">
        <span class="text-xs font-medium text-surface-600-400">Tool:</span>
        <code class="ml-2 rounded bg-primary-100-900 px-2 py-1 text-xs">
            {message.toolName}
        </code>
    </div>

    <!-- Output Toggle -->
    <button
        type="button"
        class="btn preset-outlined-primary-500 btn-sm w-full mb-2"
        onclick={() => showOutput = !showOutput}
    >
        {showOutput ? 'Hide' : 'View'} Output
    </button>

    <!-- Output Content -->
    {#if showOutput}
        <pre class="text-xs text-left mt-2 preset-filled-surface-200-800 p-3 rounded-lg max-h-96 overflow-auto font-mono whitespace-pre border border-surface-300-700">{outputContent}</pre>
    {/if}
</div>
```

### 2b. `src/lib/components/tool-messages/ObpApiResponse.svelte`

Replace the entire file with:

```svelte
<script lang="ts">
    import type { ToolMessage } from '$lib/opey/types';
    import { toast } from '$lib/utils/toastService';
    import { Copy, CheckCircle, AlertTriangle } from '@lucide/svelte';

    let { message }: { message: ToolMessage } = $props();

    let parsedOutput = $derived.by(() => {
        try {
            return typeof message.toolOutput === 'string'
                ? JSON.parse(message.toolOutput)
                : message.toolOutput;
        } catch {
            return null;
        }
    });

    let isError = $derived(
        parsedOutput?.error ||
        parsedOutput?.message ||
        (parsedOutput?.code && parsedOutput.code !== 200) ||
        (parsedOutput?.status && parsedOutput.status >= 400)
    );

    let outputContent = $derived(
        parsedOutput
            ? JSON.stringify(parsedOutput, null, 2)
            : typeof message.toolOutput === "string"
                ? message.toolOutput
                : "No output available."
    );

    let showOutput = $state(false);

    async function copyToClipboard() {
        try {
            await navigator.clipboard.writeText(outputContent);
            toast.success('Output copied to clipboard!');
        } catch (err) {
            console.error('Failed to copy: ', err);
            toast.error('Failed to copy to clipboard.');
        }
    }
</script>

<div class="card rounded-lg border-2 p-4 text-left"
     class:border-error-500={isError}
     class:bg-error-50={isError}
     class:dark:bg-error-950={isError}
     class:border-surface-300-700={!isError}
     class:bg-surface-50-950={!isError}>
    <!-- Header -->
    <div class="mb-3 flex items-center justify-between">
        <div class="flex items-center gap-2">
            {#if isError}
                <AlertTriangle class="text-error-600-400" size={20} />
                <h4 class="text-sm font-semibold text-error-700-300">API Response (Error)</h4>
            {:else}
                <CheckCircle class="text-success-600-400" size={20} />
                <h4 class="text-sm font-semibold">API Response</h4>
            {/if}
        </div>
        <button
            type="button"
            class="btn btn-sm preset-tonal-primary"
            onclick={copyToClipboard}
            title="Copy output"
            aria-label="Copy output"
        >
            <Copy size={16} />
            <span class="hidden sm:inline">Copy</span>
        </button>
    </div>

    <!-- Tool Name -->
    <div class="mb-3">
        <span class="text-xs font-medium text-surface-600-400">Tool:</span>
        <code class="ml-2 rounded bg-primary-100-900 px-2 py-1 text-xs">
            {message.toolName}
        </code>
    </div>

    <!-- Error Message -->
    {#if isError}
        <div class="mb-3 rounded-lg bg-error-100-900 p-3">
            <div class="text-sm font-medium text-error-950-50">
                {parsedOutput?.message || parsedOutput?.error || 'Request failed'}
            </div>
            {#if parsedOutput?.code || parsedOutput?.status}
                <div class="mt-1 text-xs text-error-700-300">
                    Status Code: {parsedOutput.code || parsedOutput.status}
                </div>
            {/if}
        </div>
    {/if}

    <!-- Output Toggle -->
    <button
        type="button"
        class="btn btn-sm w-full mb-2"
        class:preset-outlined-error-500={isError}
        class:preset-outlined-primary-500={!isError}
        onclick={() => showOutput = !showOutput}
    >
        {showOutput ? 'Hide' : 'View'} Full Response
    </button>

    <!-- Output Content -->
    {#if showOutput}
        <pre class="text-xs text-left mt-2 preset-filled-surface-200-800 p-3 rounded-lg max-h-96 overflow-auto font-mono whitespace-pre border border-surface-300-700">{outputContent}</pre>
    {/if}
</div>
```

### 2c. `src/lib/components/tool-messages/ToolError.svelte`

Replace the entire file with:

```svelte
<script lang="ts">
	import type { ToolMessage } from '$lib/opey/types';
	import { XCircle, AlertTriangle, Copy } from '@lucide/svelte';
	import { toast } from '$lib/utils/toastService';

	let { message }: { message: ToolMessage } = $props();

	let errorOutput = $derived(
		message.toolOutput
			? typeof message.toolOutput === 'string'
				? message.toolOutput
				: JSON.stringify(message.toolOutput, null, 2)
			: 'Tool execution failed with no details provided.'
	);

	let showDetails = $state(false);

	async function copyToClipboard() {
		try {
			await navigator.clipboard.writeText(errorOutput);
			toast.info('Error details copied to clipboard!');
		} catch (err) {
			console.error('Failed to copy: ', err);
			toast.error('Failed to copy to clipboard.');
		}
	}
</script>

<div class="card rounded-lg border-2 border-error-500 bg-error-50-950 p-4 text-left">
	<!-- Header -->
	<div class="mb-3 flex items-center justify-between">
		<div class="flex items-center gap-2">
			<XCircle class="text-error-600-400" size={24} />
			<h4 class="text-base font-semibold text-error-900-100">Tool Execution Failed</h4>
		</div>
		<button
			type="button"
			class="btn btn-sm preset-tonal-error"
			onclick={copyToClipboard}
			title="Copy error details"
			aria-label="Copy error details"
		>
			<Copy size={16} />
			<span class="hidden sm:inline">Copy</span>
		</button>
	</div>

	<!-- Tool Name -->
	<div class="mb-3">
		<span class="text-xs font-medium text-error-700-300">Tool:</span>
		<code class="ml-2 rounded bg-error-100-900 px-2 py-1 text-xs text-error-900-100">
			{message.toolName}
		</code>
	</div>

	<!-- Error Summary -->
	<div class="mb-3 flex items-start gap-2 rounded-lg bg-error-100-900 p-3">
		<AlertTriangle class="mt-0.5 flex-shrink-0 text-error-600-400" size={18} />
		<div class="text-sm text-error-900-100">
			{typeof message.toolOutput === 'string' && message.toolOutput.length < 150
				? message.toolOutput
				: 'An error occurred during tool execution. Click below to view details.'}
		</div>
	</div>

	<!-- Details Toggle -->
	<button
		type="button"
		class="btn preset-outlined-error-500 btn-sm w-full"
		onclick={() => showDetails = !showDetails}
	>
		{showDetails ? 'Hide' : 'View'} Error Details
	</button>

	<!-- Error Details -->
	{#if showDetails}
		<pre class="text-xs text-left mt-3 preset-filled-surface-200-800 p-3 rounded-lg max-h-96 overflow-auto font-mono whitespace-pre border border-error-300 dark:border-error-700">{errorOutput}</pre>
	{/if}
</div>
```

---

## 3. Config: Add missing navigation items

### `src/lib/config/navigation.ts`

Add the `FileText` icon import and "Personal Data Fields" menu item.

Update the imports:

```typescript
import { User, ShieldUser, KeyRound, IdCardLanyard, CreditCard, Database, FolderKanban, FileText } from '@lucide/svelte';
```

Add this item after the "My Data" entry in `buildMyAccountItems`:

```typescript
        { href: '/user/personal-data-fields', label: 'Personal Data Fields', iconComponent: FileText, description: 'Manage your personal attributes.' },
```

The full items array should be:

```typescript
    const items: NavigationItem[] = [
        { href: '/user', label: 'Profile', iconComponent: User },
        { href: '/user/consents', label: 'Consents', iconComponent: ShieldUser },
        { href: '/user/consumers', label: 'Consumers', iconComponent: KeyRound },
        { href: '/user/entitlements', label: 'Entitlements', iconComponent: IdCardLanyard },
        { href: '/user/my-data', label: 'My Data', iconComponent: Database, description: 'View my own data.' },
        { href: '/user/personal-data-fields', label: 'Personal Data Fields', iconComponent: FileText, description: 'Manage your personal attributes.' },
        { href: '/user/api-collections', label: 'My API Collections', iconComponent: FolderKanban, description: 'Manage your API endpoint collections.' }
    ];
```

---

## 4. Verify after applying

After all changes:

```bash
npm run check
npm run build
npm run test
```

---

## Summary of changes

| Area | Files changed | Nature |
|------|--------------|--------|
| Auth refresh | ChatService.ts, RestChatService.ts, ChatController.ts | New feature: 401 handling + callback |
| Consent events | ChatService.ts, RestChatService.ts, ChatState.ts | Add timestamp, fix optional bankId, fix operators |
| DefaultToolResponse | DefaultToolResponse.svelte | UI overhaul: card layout, Lucide icons, copy button |
| ObpApiResponse | ObpApiResponse.svelte | UI overhaul: error detection, conditional styling |
| ToolError | ToolError.svelte | UI overhaul: collapsible details, error summary |
| Navigation config | navigation.ts | Add "Personal Data Fields" menu item |
