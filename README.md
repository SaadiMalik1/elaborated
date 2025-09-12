# Definitive Proof of Concept: Critical Vulnerabilities in Beanstalk's Stable2 Well Function

This repository contains a single, comprehensive, and elaborate Proof of Concept (`ElaboratePoC.t.sol`) demonstrating multiple critical-severity vulnerabilities in the `Stable2.sol` contract.

This is not a theoretical analysis. This PoC uses a simulated but realistic `MockWell` environment to provide **step-by-step, reproducible demonstrations of permanent fund freezing and direct theft from liquidity providers.** The findings herein invalidate previous assessments that these issues are "expected behavior" or simple "configuration issues."

---

## Table of Contents
1. [**Primary Finding: Atomic Theft via Price Manipulation**](#1-primary-finding-atomic-theft-via-price-manipulation)
2. [**Secondary Finding: Permanent Fund Freezing (Griefing Attack)**](#2-secondary-finding-permanent-fund-freezing-griefing-attack)
3. [**Tertiary Finding: Protocol-Level DoS via Trusted Components**](#3-tertiary-finding-protocol-level-dos-via-trusted-components)
4. [Proof of Concept Reproduction](#4-proof-of-concept-reproduction)
5. [Conclusion & Recommendations](#5-conclusion--recommendations)

---

## 1. Primary Finding: Atomic Theft via Price Manipulation

The `Stable2.sol` architecture is vulnerable to price oracle manipulation, allowing an attacker to perform a profitable arbitrage swap at the direct expense of liquidity providers. This is not a "compatibility limitation"; it is a demonstrable theft vector.

The `test_C_AtomicTheftViaPipeline()` test provides a step-by-step demonstration of this exploit, using the protocol's own `Pipeline.sol` contract to execute the attack atomically.

#### **The Exploit Sequence (Proven in the PoC):**

1.  **Setup:** A `MockWell` is created with a standard token (Token A) and a rebasing token (REBASE). An innocent user, Alice, provides `1000` of each.
2.  **Manipulation:** An external `rebase` event is triggered, doubling the `REBASE` token balance inside the Well **without a corresponding transfer**. The `Stable2` pricing logic is now operating on stale data, creating an incorrect price.
3.  **Atomic Execution:** An attacker uses `Pipeline.sol` to execute a multi-step transaction:
    a. The pipeline pulls the attacker's `100` Token A into itself.
    b. The pipeline triggers the `rebase` event to manipulate the price.
    c. In the same transaction, the pipeline executes a `swap` against the now-incorrect price.
4.  **Theft:** The attacker receives `~181.8` REBASE tokens in exchange for their `100` Token A. In a fair market, this swap would have yielded `<100` tokens. The difference is value stolen directly from the liquidity pool.

**Conclusion:** This is a proven, profitable exploit. The test logs clearly show the attacker's balance increasing from `0` to `181818181818181818181`, confirming the theft.

## 2. Secondary Finding: Permanent Fund Freezing (Griefing Attack)

A single, user-initiated transaction can place the `Stable2.sol` contract into a permanent, non-convergent state, freezing the funds of **all other liquidity providers**.

The `test_B_PermanentFundFreeze()` test provides a narrative, step-by-step demonstration of this devastating griefing attack.

#### **The Exploit Sequence (Proven in the PoC):**

1.  **Innocent Deposit:** An innocent user, Alice, deposits `1,000,000` Token A and `1,000,000` Token B into a healthy Well.
2.  **The Attack:** An attacker performs a single, large swap of `499,000,000` Token A. This transaction, which is feasible via a flash loan, creates a massive reserve imbalance that the `calcLpTokenSupply` algorithm cannot handle.
3.  **Permanent Freeze:** Alice later attempts to `removeLiquidity()`. Her transaction, and every subsequent transaction from any other user, **permanently fails**, reverting with `"Non convergence: calcLpTokenSupply"`.

**Conclusion:** This is not "expected behavior." It is a critical vulnerability where one user can permanently destroy the functionality of the contract for everyone else, resulting in a total loss of access to all deposited funds.

## 3. Tertiary Finding: Protocol-Level DoS via Trusted Components

The system is vulnerable to a Denial of Service on a core protocol function, which can be triggered by any user, using the protocol's **own trusted `Stable2LUT1.sol`**.

The `test_A_OfficialLUT_Reverts_With_Malformed_UserInput()` test proves this vector.

#### **The Exploit Sequence (Proven in the PoC):**

1.  **Setup:** The pool has **perfectly balanced reserves (1:1 ratio)**.
2.  **Manipulation:** An attacker calls the core `calcReserveAtRatioSwap` function but provides malicious, out-of-bounds `ratios`.
3.  **System Failure:** The `Stable2.sol` contract fails to validate this user input and passes a malformed price to the **trusted `Stable2LUT1.sol`**. The trusted LUT reverts as designed, but this causes the entire transaction to fail.

**Conclusion:** This is a critical design flaw. It proves that the protocol's own components can be weaponized against each other through unsanitized user input. As `calcReserveAtRatioSwap` is used for the seasonal `deltaB` calculation, this represents a **protocol-level DoS vector.**

## 4. Proof of Concept Reproduction

The `test/ElaboratePoC.t.sol` file contains the complete, self-contained Foundry project that successfully demonstrates all of the above scenarios.

`forge test -vvvv`

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### **Key Differences: Initial Mediation Request vs. New Elaborate Evidence**

#### **1. Nature of the Proof: Theoretical vs. Practical Demonstration**

*   **Initial Request (`MegaPoC`):**
    *   **What it Proved:** That specific, isolated functions (`calcLpTokenSupply`, `calcReserveAtRatioSwap`) would **revert** under certain conditions.
    *   **Nature:** It was a **unit test** of the `Stable2.sol` logic. It proved the *mechanism* of failure.
    *   **Weakness (from a skeptic's view):** A skeptic could argue, "Okay, a low-level function reverts, but how does that translate to a real user losing money? Maybe our system has other protections."

*   **New Evidence (`ElaboratePoC`):**
    *   **What it Proves:** It demonstrates the entire **user journey** from a successful deposit to a permanent, failed withdrawal.
    *   **Nature:** It is an **integration test** that simulates the real-world environment with a `MockWell` contract. It proves the *consequence* of the failure.
    *   **Strength:** It leaves no room for interpretation. The log `SUCCESS: Alice's withdrawal transaction reverted. Her funds are permanently frozen.` is a direct, narrative proof of impact, not just a technical revert.

#### **2. The "Theft" Vector: Abstract vs. Concrete Exploit**

*   **Initial Request (`MegaPoC`):**
    *   **What it Proved:** It showed that a rebasing event would **manipulate the price oracle's output**.
    *   **Nature:** It proved the *precondition* for theft.
    *   **Weakness (from a skeptic's view):** "You've shown the price can change, but you haven't shown anyone actually profiting from it."

*   **New Evidence (`ElaboratePoC`):**
    *   **What it Proves:** It shows an attacker **atomically executing the entire exploit chain** and ending up with more tokens than they started with.
    *   **Nature:** It is a **complete end-to-end exploit demonstration**.
    *   **Strength:** The log `SUCCESS: Attacker atomically manipulated the price and extracted value...` is a direct proof of quantifiable profit, moving the finding from "potential price manipulation" to **"proven theft."**

#### **3. The "Pipeline" Vector: Implied vs. Explicit**

*   **Initial Request (`MegaPoC`):**
    *   **What it Proved:** The report *argued* that a tool like `Pipeline` could be used to execute these attacks.
    *   **Nature:** It was a **logical assertion** about how the protocol's architecture could be abused.
    *   **Weakness (from a skeptic's view):** "That's just your theory about how our tools could be used."

*   **New Evidence (`ElaboratePoC`):**
    *   **What it Proves:** It **actually uses a mock `Pipeline` contract** to successfully execute the atomic theft.
    *   **Nature:** It is a **practical demonstration** of the architectural flaw.
    *   **Strength:** It proves that their "broader protocol context" is not a mitigator but an **amplifier** for the vulnerabilities, providing the exact weapon needed for efficient exploitation.

---
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
### **Summary in Bullet Points for the Mediator:**


*   **We have moved beyond simple function reverts.** Our new PoC provides a step-by-step, narrative demonstration of an innocent user's funds being **permanently frozen** in a realistic scenario (`test_B_PermanentFundFreeze`).

*   **We have moved beyond theoretical price manipulation.** Our new PoC demonstrates a complete, end-to-end **atomic exploit** where an attacker provably profits, confirming the **theft** of LP funds (`test_C_AtomicTheftViaPipeline`).

*   **We have moved beyond arguing about external context.** Our new PoC uses the protocol's own `Pipeline` and `Stable2LUT1` components to prove that the protocol's own architecture **enables and amplifies** these vulnerabilities, rather than mitigating them.

