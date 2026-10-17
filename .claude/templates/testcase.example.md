# Test case — <Feature name>

Source: requirements/<system>/<feature>.md
Feature: `tests/<feature>/` · tag `@<feature>`

| TC-ID | TC Description | Test data/ Pre-condition | Steps | Expected Result | Automation | Result | Notes |
|---|---|---|---|---|---|---|---|
| TC-<FEATURE>-001 | <description> | <data> | B1: <step 1><br>B2: <step 2> | <expected result> | ✅ | Not run | `@smoke` |

Total: <n> cases · Automated <n> · Manual <n>
From brief: <n> · AI-suggested: <n>

---

**Multi-screen feature** (has a "Screens in flow" table in requirements):
- Declare SUBFLOWs at the top of the file (inherited from requirements) so the whole team uses consistent names.
- End-to-end happy path uses a code with no SUBFLOW (TC-<FEATURE>-001).
- Per-screen edge cases use TC-<FEATURE>-<SUBFLOW>-001.

<!-- Declare screen SUBFLOWs — adjust per feature:
  SUBFLOW CART   → tests/checkout/cart.spec.ts          · range TC-CHECKOUT-CART-001..099
  SUBFLOW ADDR   → tests/checkout/address.spec.ts       · range TC-CHECKOUT-ADDR-001..099
  SUBFLOW PAY    → tests/checkout/payment.spec.ts       · range TC-CHECKOUT-PAY-001..099
-->

### Happy path (end-to-end)

| TC-ID | TC Description | Test data/ Pre-condition | Steps | Expected Result | Automation | Result | Notes |
|---|---|---|---|---|---|---|---|
| TC-<FEATURE>-001 | Go from screen 1 to last screen with valid input | <data> | B1: <step 1><br>B2: <step 2> | Flow complete, reaches result screen | ✅ | Not run | `main-flow.spec.ts` · `@smoke` |

### Screen 1: <screen name> (SUBFLOW: <ABC>)

| TC-ID | TC Description | Test data/ Pre-condition | Steps | Expected Result | Automation | Result | Notes |
|---|---|---|---|---|---|---|---|
| TC-<FEATURE>-<ABC>-001 | <edge case for screen 1> | <data> | B1: <step 1><br>B2: <step 2> | <expected result> | ✅ | Not run | `@regression` |

### Screen 2: <screen name> (SUBFLOW: <DEF>)

| TC-ID | TC Description | Test data/ Pre-condition | Steps | Expected Result | Automation | Result | Notes |
|---|---|---|---|---|---|---|---|
| TC-<FEATURE>-<DEF>-001 | <edge case for screen 2> | <data> | B1: <step 1><br>B2: <step 2> | <expected result> | ✅ | Not run | Enter via "Direct entry" · `@regression` |

Total: <n> cases · Automated <n> · Manual <n> · Across <k> screens
From brief: <n> · AI-suggested: <n>
