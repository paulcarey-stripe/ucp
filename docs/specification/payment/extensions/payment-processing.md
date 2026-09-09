<!--
   Copyright 2026 UCP Authors

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

# Payment Processing Extension

* **Capability Name:** `dev.ucp.common.payment.processing`
* **Maturity:** Working Draft

> **Working Draft:** This extension is intended for prototyping and feedback.
> Breaking changes are expected before Candidate maturity.

## Overview

The Payment Processing extension lets the Platform request a capture policy
before Checkout completion and lets the Business report authorization and
capture activity on the resulting Order. It does not define operations for the
Platform to invoke. The Business and its provider perform the payment work.

Processing is organized into four parts:

* An **approval** records a maximum amount and capture policy agreed by the
  Buyer.
* A **capture policy** records when capture should start and whether more than
  one capture is planned.
* An **attempt** describes one provider processing lifecycle for an instrument.
* An **entry** records one processing event and is not changed after it is
  reported.

Order instruments continue to provide the simpler authorization and capture
summary. This extension adds provider processing detail without returning
credentials or other sensitive payment data.

## Dependencies and Amounts

This extension requires Checkout and Order versions that define Checkout amount
finality, the maximum amount approved by the Buyer, and Order payment
instruments.

| Value | Meaning |
| --- | --- |
| Instrument `authorized_amount` | Authorization currently effective for the instrument. |
| Instrument `captured_amount` | Gross successful capture or collection before adjustments and capture reversals. |
| Approval `maximum_amount` | Maximum total payment amount approved by the Buyer at that point. |
| Attempt `capturable_amount` | Current capture capacity reported by the provider, when known. |

The Checkout approval's maximum equals `order.payment.maximum_amount`. If the
Buyer later approves an increase, the Business adds another approval whose
`maximum_amount` is the new total ceiling, not the amount of the increase.

Before requesting an authorization, authorization adjustment, or capture, the
Business MUST ensure that it would not authorize or collect more for the Order
than the `maximum_amount` in the approval it records on the entry. Historical
entries cannot be added together for this check because they include expired or
replaced authorizations and reversed captures.

Interaction with the Split Payments extension, including how a later increase
is allocated across instruments, is outside this Working Draft.

All reported amounts use the Order currency.

## Capability Negotiation

Businesses advertise the extension with both parent capabilities:

<!-- ucp:example skip reason="illustrative capability declaration fragment" -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "capabilities": {
      "dev.ucp.common.payment.processing": [
        {
          "version": "{{ ucp_version }}",
          "extends": [
            "dev.ucp.shopping.checkout",
            "dev.ucp.shopping.order"
          ],
          "spec": "https://ucp.dev/{{ ucp_version }}/specification/payment/extensions/payment-processing",
          "schema": "https://ucp.dev/{{ ucp_version }}/schemas/common/payment_processing.json",
          "config": {
            "capture": {
              "modes": ["automatic", "manual"],
              "sequences": ["single", "multiple"],
              "max_captures": 50
            }
          }
        }
      ]
    }
  }
}
```

<!-- ucp:example schema=common/payment_processing def=business_config -->
```json
{
  "capture": {
    "modes": ["automatic", "manual"],
    "sequences": ["single", "multiple"],
    "max_captures": 50
  }
}
```

Configuration states which policies a participant supports. This extension
defines no payment operations for the Platform to call. `max_captures` is
optional and does not override provider or payment-method eligibility. Capture
mode and sequence values are open; a participant must understand a value before
selecting it.

## Selecting a Capture Policy

The Platform MAY request a supported capture policy on Create or Update
Checkout. When this extension is active, the Business MUST return the selected
policy in every Checkout response. If the Platform cannot accept the returned
policy, it MUST update the Checkout again or hand off before completion.

<!-- ucp:example schema=common/payment_processing def=checkout_processing -->
```json
{
  "capture": {
    "mode": "manual",
    "sequence": "multiple"
  }
}
```

`automatic` means the Business plans to start capture as part of completing the
Checkout. The result may arrive later and need not appear in the completion
response. `manual` means the Business plans to start capture later through its
own workflow.

The `complete_checkout` request omits this field, so the Platform cannot change
the policy during completion. Completion uses the policy already selected on
the Checkout, allowing the Platform to explain the timing before the Buyer
commits.

Once Complete Checkout is accepted, its capture policy MUST NOT change. The
Checkout approval on the Order MUST use that policy. A later Order change may
add an approval with a different policy. Entries record what actually happened,
which may differ from the selected policy.

## Order Reporting

Every Order report contains exactly one approval with `type: "checkout"`. Its
`maximum_amount` equals `order.payment.maximum_amount`, and its capture policy
matches the completed Checkout. A later approval uses `type: "order_change"`,
references the corresponding Order adjustment, and records the new total
ceiling.

Approvals and processing attempts are separate collections. Each attempt
refers to an existing `order.payment.instruments[].id`, and multiple attempts
may refer to one instrument. An attempt stays intact when a later approval
allows the Business to increment an existing authorization. Its subsequent
entries reference the new approval.

<!-- ucp:example schema=common/payment_processing def=order_processing -->
```json
{
  "approvals": [
    {
      "id": "approval_checkout_01",
      "type": "checkout",
      "maximum_amount": 10000,
      "capture": {
        "mode": "manual",
        "sequence": "single"
      }
    }
  ],
  "attempts": [
    {
      "id": "pa_card_01",
      "instrument_id": "pi_card_01",
      "status": "captured",
      "captured_amount": 9500,
      "capturable_amount": 0,
      "entries_complete": true,
      "entries": [
        {
          "id": "pae_auth_01",
          "type": "authorization",
          "status": "succeeded",
          "approval_id": "approval_checkout_01",
          "requested_amount": 10000,
          "amount": 9500,
          "observed_at": "2026-09-08T14:00:00Z",
          "occurred_at": "2026-09-08T13:59:59Z",
          "expires_at": "2026-09-15T14:00:00Z"
        },
        {
          "id": "pae_capture_01",
          "type": "capture",
          "status": "succeeded",
          "approval_id": "approval_checkout_01",
          "requested_amount": 9500,
          "amount": 9500,
          "final": true,
          "authorization_entry_id": "pae_auth_01",
          "observed_at": "2026-09-08T14:05:00Z"
        },
        {
          "id": "pae_release_01",
          "type": "authorization_release",
          "status": "succeeded",
          "authorization_entry_id": "pae_auth_01",
          "observed_at": "2026-09-08T14:05:01Z"
        }
      ]
    }
  ]
}
```

The example records a partial authorization: 100.00 was requested and 95.00
approved. The final capture collected 95.00 and released the authorization, so
the attempt has no current `authorized_amount`.

### Full Order Example

<!-- ucp:example schema=common/payment_processing def=dev.ucp.shopping.order op=read direction=response -->
```json
{
  "ucp": {
    "version": "{{ ucp_version }}",
    "status": "success",
    "capabilities": {
      "dev.ucp.shopping.order": [{"version": "{{ ucp_version }}"}],
      "dev.ucp.common.payment.processing": [{"version": "{{ ucp_version }}"}]
    }
  },
  "id": "order_processing_01",
  "checkout_id": "checkout_processing_01",
  "permalink_url": "https://business.example.com/orders/processing_01",
  "line_items": [],
  "fulfillment": {},
  "currency": "USD",
  "totals": [
    {"type": "subtotal", "amount": 9500},
    {"type": "total", "amount": 9500}
  ],
  "payment": {
    "maximum_amount": 10000,
    "instruments": [
      {
        "id": "pi_card_01",
        "type": "card",
        "captured_amount": 9500,
        "display": {"brand": "visa", "last_digits": "4242"}
      }
    ],
    "processing": {
      "approvals": [
        {
          "id": "approval_checkout_01",
          "type": "checkout",
          "maximum_amount": 10000,
          "capture": {"mode": "manual", "sequence": "single"}
        }
      ],
      "attempts": [
        {
          "id": "pa_card_01",
          "instrument_id": "pi_card_01",
          "status": "captured",
          "captured_amount": 9500,
          "capturable_amount": 0,
          "entries_complete": false,
          "entries": []
        }
      ]
    }
  }
}
```

## Approvals

Approvals are immutable and appear in the order the Buyer gave them. The first
approval has `type: "checkout"`. Its maximum and capture policy come from the
completed Checkout.

When an Order change increases the amount, a new approval records the revised
total ceiling and references the corresponding `order.adjustments` entry. For
example, if the Checkout approval was 100.00 and the Buyer later approves an
additional 20.00, the new approval's `maximum_amount` is 12000. The earlier
approval remains unchanged.

<!-- ucp:example schema=common/payment_processing def=payment_approval -->
```json
{
  "id": "approval_change_01",
  "type": "order_change",
  "maximum_amount": 12000,
  "capture": {
    "mode": "manual",
    "sequence": "single"
  },
  "adjustment_id": "adjustment_add_item_01"
}
```

When a provider increases an existing authorization after an Order change, the
Business appends the new approval to `approvals[]` and an
`authorization_adjustment` to the existing attempt's `entries[]`. The original
authorization entry remains unchanged. The new entry references both the
original authorization and the new approval, while the attempt's
`authorized_amount` is updated to the current amount.

<!-- ucp:example schema=common/payment_processing def=processing_entry -->
```json
{
  "id": "pae_increment_01",
  "type": "authorization_adjustment",
  "status": "succeeded",
  "approval_id": "approval_change_01",
  "authorization_entry_id": "pae_auth_01",
  "requested_amount": 12000,
  "amount": 12000,
  "observed_at": "2026-09-09T10:00:00Z"
}
```

## Attempts

An attempt is one provider processing lifecycle for one Order instrument. A
transport retry with the same idempotency key does not create another attempt.
Incrementing or reducing an existing authorization remains in the same attempt.
A replacement authorization or provider failover that starts another provider
lifecycle receives a new attempt ID. An attempt has at most one root
authorization.

The attempt summary describes its current state:

* `authorized_amount` is the authorization currently effective in the attempt.
  It MUST be present when that amount is greater than zero; omission means no
  authorization remains effective.
* `captured_amount` is the total successfully captured before refunds, credits,
  disputes, or capture reversals. It MUST be present after a successful
  capture; omission means nothing has been captured.
* `capturable_amount` is provider-reported current capacity and is never derived.
* `expires_at` is the current provider-reported authorization expiry.

The parent instrument's `authorized_amount` equals the sum of
`authorized_amount` across its attempts. Its `captured_amount` likewise equals
the sum of `captured_amount` across those attempts. The Business MUST keep the
instrument and attempt summaries consistent.

The Business reports the current attempt status. `captured` means it considers
capture complete for that attempt; `partially_captured` means another capture
may follow. Consumers MUST accept unknown status values and continue to use the
reported amount summaries.

## Entries

Known entry types have these semantics:

| Type | Amount behavior |
| --- | --- |
| `authorization` | `requested_amount` is the requested authorization; successful `amount` is the approved amount. |
| `authorization_adjustment` | References a root authorization. Requested and successful amounts are new totals, never deltas; this supports increases and decreases. |
| `capture` | Requested and successful amounts are reported separately. `final` is true when the Business intends no further captures for the attempt. |
| `capture_reversal` | References a successful capture. Requested and successful amounts report the portion voided before settlement. |
| `authorization_release` | Fully releases the referenced authorization. A partial reduction is an adjustment. |
| `authorization_expiry` | Records expiry of the referenced authorization. |

Type and status are open vocabularies. The schema enforces known local shapes
without rejecting future values.

Authorization, authorization-adjustment, and capture entries include an
`approval_id`. It MUST match an approval in the same Order. A terminal entry
that resolves a pending request uses the same approval. Failed entries include
a Business-defined `failure_code` and may include an explanation that is safe
to disclose.

### Pending Resolution

A timeout is unresolved, not failed. A pending authorization, adjustment,
capture, or reversal carries the requested amount and `observed_at`.
`occurred_at` is omitted when the provider time is unknown.

<!-- ucp:example schema=common/payment_processing def=processing_entry -->
```json
{
  "id": "pae_auth_pending",
  "type": "authorization",
  "status": "pending",
  "approval_id": "approval_checkout_01",
  "requested_amount": 10000,
  "observed_at": "2026-09-08T14:00:00Z"
}
```

After the Business reports a pending entry, it MUST NOT change that entry. To
resolve it, the Business appends a terminal entry of the same type. The terminal
entry references the pending entry and carries the actual result, which may
differ from the request:

<!-- ucp:example schema=common/payment_processing def=processing_entry -->
```json
{
  "id": "pae_auth_succeeded",
  "type": "authorization",
  "status": "succeeded",
  "approval_id": "approval_checkout_01",
  "amount": 9500,
  "resolves_entry_id": "pae_auth_pending",
  "observed_at": "2026-09-08T14:00:05Z",
  "occurred_at": "2026-09-08T14:00:03Z"
}
```

The same pattern applies to a pending authorization release, but no amount is
reported for a full release. If the pending entry was never delivered, the
Business may report the terminal entry directly. A delivered pending entry
MUST NOT be updated in place and may have at most one terminal entry that
references it through `resolves_entry_id`.

### Reconciliation

For each root authorization:

1. A successful authorization `amount` establishes its current effective amount.
2. A successful adjustment replaces that amount with the adjustment `amount`.
3. Release or expiry makes its current effective amount zero.
4. Capture does not itself define the remaining authorization or capturable amount.

Successful captures accumulate in the attempt and instrument
`captured_amount`. A `capture_reversal` records a pre-settlement void and does
not rewrite that gross historical amount. Refunds, credits, disputes, and
commercial cancellation remain Order adjustments.

Consumers MUST NOT derive capturable capacity by subtracting authorization and
capture, and MUST NOT reject capture solely because it exceeds current
authorization. An amount captured above the current authorization is still
subject to the referenced approval's `maximum_amount`, provider eligibility,
network rules, and law.

## Updating and Retaining Entries

Entries appear in the order the Business observes them, which may differ from
provider occurrence time. Once reported, an entry MUST keep the same content
and relative order. Receiving the same entry again does not change the report.

A Business does not have to retain the full history on the Order. Under its
documented retention policy, it MAY remove terminal entries from the beginning
of the array. It MUST NOT remove a pending entry, an entry referenced by a
retained entry, or an entry between retained entries. After removing any entry,
the Business MUST report `entries_complete: false`. Consumers MUST continue to
use the current summary fields when older entries are absent. Retrieving
discarded history is outside this Working Draft.

| Condition | Required behavior |
| --- | --- |
| Timeout or ambiguous result | Report pending and reconcile later. |
| Definitive rejection | Report a failed entry without changing earlier entries. |
| Duplicate or out-of-order delivery | Preserve entry identity and observation order. |
| Late success | Append a terminal entry and update summaries. |

## Security and Authority

The Buyer or Platform may receive this Order data. `reference` MAY contain an
opaque provider identifier used for reconciliation. It MUST NOT contain a PAN,
CVV, bank account number, credential, bearer secret, reusable token, Checkout
routing data, or anything that could be used to make another payment.
`failure_message` MUST NOT contain a raw provider response or other sensitive
data.

Platform-facing 3DS device data collection and challenge surfaces are defined
by the Payment Authentication extension. Provider authentication outcomes are
outside this extension. Settlement, payout, refunds, disputes, and fulfillment
also remain outside it. A future extension that defines payment operations for
the Platform would need its own negotiated capability and authority rules.

When AP2 is active, the signed Checkout includes the selected capture policy,
`amount_finality`, and `maximum_amount`. For a provisional total, the mandate
records the maximum approved by the Buyer. A higher amount requires separate
Buyer approval, recorded separately from the provider attempt.

## Schema

### Business Config

{{ extension_schema_fields('payment_processing.json#/$defs/business_config', 'payment/extensions/payment-processing') }}

### Platform Config

{{ extension_schema_fields('payment_processing.json#/$defs/platform_config', 'payment/extensions/payment-processing') }}

### Capture Config

{{ extension_schema_fields('payment_processing.json#/$defs/capture_config', 'payment/extensions/payment-processing') }}

### Capture Policy

{{ extension_schema_fields('payment_processing.json#/$defs/capture_policy', 'payment/extensions/payment-processing') }}

### Capture Mode Value

{{ extension_schema_fields('payment_processing.json#/$defs/capture_mode_value', 'payment/extensions/payment-processing') }}

### Capture Sequence Value

{{ extension_schema_fields('payment_processing.json#/$defs/capture_sequence_value', 'payment/extensions/payment-processing') }}

### Checkout Processing

{{ extension_schema_fields('payment_processing.json#/$defs/checkout_processing', 'payment/extensions/payment-processing') }}

### Order Processing

{{ extension_schema_fields('payment_processing.json#/$defs/order_processing', 'payment/extensions/payment-processing') }}

### Payment Approval

{{ extension_schema_fields('payment_processing.json#/$defs/payment_approval', 'payment/extensions/payment-processing') }}

### Processing Attempt

{{ extension_schema_fields('payment_processing.json#/$defs/processing_attempt', 'payment/extensions/payment-processing') }}

### Attempt Status

{{ extension_schema_fields('payment_processing.json#/$defs/attempt_status', 'payment/extensions/payment-processing') }}

### Processing Entry

{{ extension_schema_fields('payment_processing.json#/$defs/processing_entry', 'payment/extensions/payment-processing') }}

### Entry Type

{{ extension_schema_fields('payment_processing.json#/$defs/entry_type', 'payment/extensions/payment-processing') }}

### Entry Status

{{ extension_schema_fields('payment_processing.json#/$defs/entry_status', 'payment/extensions/payment-processing') }}
