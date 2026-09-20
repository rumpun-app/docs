# Payment and subscriptions

Base: `/api/v1`. Route namespace: `/payment/*`. Phase: active MVP v1.1. First provider is DOKU through a provider-neutral adapter.

The Payment module owns payment accounts, the versioned catalog, hosted checkout sessions, payment orders and attempts, subscriptions, internal invoices, entitlement grants, refunds, webhook inbox records, dunning, and reconciliation. **Payment never owns membership, Family Keys, plaintext family content, or recovery rights.**

## Operations

| Method | Path | Operation |
|---|---|---|
| POST | `/payment/accounts` | `createPaymentAccount` |
| GET | `/payment/accounts/{payment_account_id}` | `getPaymentAccount` |
| GET | `/payment/catalog` | `getPaymentCatalog` |
| POST | `/payment/checkout-sessions` | `createPaymentCheckoutSession` |
| GET | `/payment/checkout-sessions/{checkout_session_id}` | `getPaymentCheckoutSession` |
| POST | `/payment/subscriptions/{subscription_id}/cancel` | `cancelPaymentSubscription` |
| POST | `/payment/subscriptions/{subscription_id}/resume` | `resumePaymentSubscription` |
| POST | `/payment/subscriptions/{subscription_id}/change-plan` | `changePaymentSubscriptionPlan` |
| GET | `/payment/subscriptions/current` | `getCurrentPaymentSubscription` |
| GET | `/payment/invoices` | `listPaymentInvoices` |
| GET | `/payment/invoices/{invoice_id}` | `getPaymentInvoice` |
| POST | `/payment/payments/{payment_id}/refund-requests` | `createPaymentRefundRequest` |
| GET | `/payment/entitlements` | `getPaymentEntitlements` |
| POST | `/webhooks/payments/{provider}` | `receivePaymentProviderWebhook` |

## Non-negotiable rules

- **Provider-hosted checkout only.** Rumpun never receives PAN, CVV, PIN, OTP, raw wallet tokens, or bank credentials.
- A browser redirect means processing only. Plans activate **after a verified webhook or server-to-server status lookup**, never on redirect alone.
- All mutations and webhook deliveries are idempotent.
- Money is integer minor units, **IDR only** for MVP.
- Payment and family access are separate domains. Downgrade, failed payment, suspension, cancellation, refund, or expiry never deletes ciphertext or keys.
- Entitlements may limit new growth but cannot weaken E2EE, basic recovery, lawful deletion, or legitimate read access.
- Only the payment owner sees method-specific failure details.

## Webhooks

`POST /webhooks/payments/{provider}` receives provider callbacks. Deliveries are verified and idempotent, applied in order, and recorded in a webhook inbox. Provider acceptance, retry, bounce, and dead-letter handling are internal domain contracts, not public user APIs.

## Stable failures

Payment operations use the shared error envelope and stable codes (see [Errors](errors.md)), including the ADR-091 authorization-context codes. Payment-account isolation is enforced as a tenancy boundary: foreign payment accounts are concealed, and a mismatch returns `RESOURCE_TENANCY_MISMATCH`.
