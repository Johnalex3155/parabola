# Failure Modes and Recovery Guide

This guide provides a comprehensive framework for handling failures during the USDC transfer process between Arc and Stellar. Because transfers involve locking funds on a source chain (the "Burn"), failure handling is critical to ensure funds are never lost and the transfer is eventually completed.

## 🔄 The Transfer Lifecycle
Every transfer follows the **Burn $\rightarrow$ Attest $\rightarrow$ Mint** flow. A failure can occur at any of these three stages:

1.  **Burn (Source Chain):** USDC is locked/burned on the source chain.
2.  **Attestation (Circle Iris):** Circle observes the burn and, once the source chain reaches the required finality threshold, signs an attestation.
3.  **Mint (Destination Chain):** The attested message is submitted to the destination chain: `receiveMessage` on Arc, or for Stellar, `mint_and_forward` on Circle's `CctpForwarder` contract, which mints to itself and then forwards to the real recipient. Direct minting to a Stellar address is not supported by CCTP, which is why the forwarder step exists.

---

## 🛠 The Failure Matrix

| Error Class | Meaning | Fund Status | Recommended Action |
| :--- | :--- | :--- | :--- |
| **`TransferError`** | Burn succeeded, but attestation polling or minting failed. `burnTxHash` identifies the recoverable burn; `attestationHash` is present if attestation was already retrieved. | **Locked on Source** | Persist the hashes. Wait if `code === \"ATTESTATION_NOT_READY\"`; otherwise diagnose the cause, then resume with `completeMint()`. Do not burn again. |
| **`AttestationNotReadyError`** | Burn is confirmed, but Circle has not signed the attestation yet. | **Safe (Locked)** | Retry later. Typical for new burns. |
| **`IrisRequestError`** | The Circle-side lookup failed; it does not establish that the burn or mint failed. `status` contains the HTTP status when available. | **Safe (Locked)** | Treat as upstream outage. Back off and retry lookup. Preserve the burn hash. |
| **`SubmissionTimeoutError`** | Transaction was broadcast, but confirmation was not observed before timeout. | **Unknown** | **Inspect first.** Check the hash on-chain before retrying. Never assume nothing was submitted. |

---

## 📉 Recovery Scenarios

### Scenario 1: Failure BEFORE the Burn
If an error occurs before the source chain transaction is confirmed, no funds have left the wallet.
- **Action:** Simply retry the `transfer()` call.

### Scenario 2: Failure AFTER Burn, BEFORE Attestation
The source funds are locked, but Circle's Iris service hasn't signed the message yet.
- **Action:** Do NOT burn again. Persist the `burnTxHash`. Poll using `getAttestation()` or `resolveBurn()` until the attestation is ready.

### Scenario 3: Failure AFTER Attestation, BEFORE Mint
The attestation is ready, but the final minting transaction failed or timed out.
- **Action:** Use `completeMint()`. Since you already have the `burnTxHash`, you can skip the burn/attest steps and go straight to the mint.

---

## 💻 Professional Recovery Implementation

For long-running services, we recommend a durable recovery loop. This example demonstrates how to handle a `TransferError` by recording the burn and resuming later.

```typescript
import { completeMint, transfer, TransferError } from "@asctp/parabola";

async function durableTransfer(params) {
  try {
    // Attempt the full flow
    const result = await transfer(params);
    await ledger.markComplete(result.burnTxHash, result.mintTxHash);
    return result;
  } catch (error) {
    if (!(error instanceof TransferError)) throw error;

    // CRITICAL: The burn succeeded, but the process stopped.
    // Record the burn hash immediately to prevent fund loss.
    await ledger.record({
      burnTxHash: error.burnTxHash,
      network: params.network ?? "mainnet",
      params: params
    });

    // Trigger asynchronous recovery
    scheduleRecovery(error.burnTxHash);
    throw new Error(`Transfer pending recovery. BurnHash: ${error.burnTxHash}`);
  }
}

async function recoverTransfer(burnTxHash) {
  const record = await ledger.get(burnTxHash);
  
  try {
    const recovered = await completeMint({
      from: record.params.from,
      to: record.params.to,
      burnTxHash: burnTxHash,
      network: record.network,
      signer: record.destinationSigner,
    });
    
    await ledger.markComplete(burnTxHash, recovered.mintTxHash);
  } catch (err) {
    // Back off and retry based on error type (e.g., AttestationNotReadyError)
  }
}
```

---

## ⚙️ Tuning & Optimization

### 1. `pollTimeout` vs. `pollInterval`
- **`pollTimeout` (Default: 300s):** The maximum time a single SDK call waits for attestation. If you have a latency-bounded API, set this shorter and handle the timeout by persisting the burn hash and resuming asynchronously.
- **`pollInterval` (Default: 3s):** How often the SDK checks Iris. Shorter intervals do not make Circle attest faster; they only increase API traffic.

### 2. Stellar RPC Tuning
If using a slow Soroban RPC provider, increase the retry limits in `options`:
- **`stellarRetryAttempts`**: Increase for slower convergence of allowance reads.
- **`stellarRetryDelayMs`**: Increase the delay between retries.

---

## ⚠️ The Idempotency Warning
`completeMint()` does **not** deduplicate. If you call it twice for the same burn, you will pay gas for the second transaction only to have the chain reject it. 

**Rule:** Always keep a local ledger keyed by `(burnTxHash, network)`. Before calling `completeMint()`, check if you already have a recorded `mintTxHash` for that burn.
