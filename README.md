<!-- solve-problem:5dcee270-8c55-4c61-b7fc-77471343f9b6 -->
# Pricing persistent-state externalities

A [Tangle Solve problem](https://solve.tangle.tools/problems/5dcee270-8c55-4c61-b7fc-77471343f9b6).

## What is the problem?

In blockchain fee markets, a transaction pays gas priced against its *immediate* computational cost, but transactions also mutate persistent state (creating accounts, writing storage slots, growing the UTXO set or contract storage). Each byte of live state imposes a perpetual obligation on every future full node: it must be stored, merklized, and served, forever. The problem is to design a fee mechanism that charges transactions for the *intertemporal* burden they impose, in a way that is:

1. **Manipulation-resistant:** no profitable splitting (one large write split into many small writes that individually fall below the pricing threshold), no profitable create–delete–recreate cycles (e.g., in Ethereum, deleting storage refunds gas via `SSTORE` refunds, then recreating it — any naive "state growth fee" must not be farmed through refund games);
2. **Access-preserving:** the mechanism must not make touching existing state so expensive that ordinary contract execution and cold/warm access patterns become economically unusable;
3. **Incentive-compatible / near-optimal:** in the formalism of multidimensional fee markets, the mechanism should approximately clear the market for the "state-size resource" alongside execution, bandwidth, and calldata, with bounded social welfare loss.

Formal setting (most reasonable interpretation): discrete-time block building; transactions arrive with multidimensional resource vectors including a component ΔS (net state growth, possibly negative with deletions); the mechanism sets dynamic prices p_t ∈ R^k per resource (e.g., via an EIP-1559-style target/feedback rule or an auction). The goal is a pricing rule and refund policy such that (a) the long-run state size converges to a target under adversarial users, and (b) the mechanism is collusion-proof against splitting, since "state" is not a fungible, separable resource — this is precisely where the Angeris–Diamandis–Moallemi near-optimality framework needs extension, because their optimality relies on convex-separable resource costs, and persistent state cost is neither (it depends on the full state history, not the transaction alone).

## Why does it matter?

Ethereum's state has grown to hundreds of GBs, driving up node hardware requirements and centralizing validation; Bitcoin's UTXO set creates the same externality. Solving this unlocks economically sound state-growth control (replacing blunt instruments like EIP-4444 history expiry or rent proposals that never shipped), extends the 2024 optimality theory of multidimensional fees to non-separable, intergenerational resources, and would provide the formal foundations regulators/protocol designers need for durable-storage cost sharing analogous to environmental externality pricing (e.g., deposit-refund systems).

## What's already known?

- **Angeris, Diamandis, Moallemi (2024)** prove that multidimensional EIP-1559-style fees (separate floating prices per resource with target tracking) are essentially optimal — within constant factors of the ideal auction — *when resource costs are convex and separable across transactions in a block*. Persistent state violates this: the marginal cost of a write depends on the current state size and on whether the slot is later deleted.
- Ethereum practice: `SSTORE` gas tiers (cold/warm/create/modify) plus refund mechanics attempt ad hoc state pricing; the refund design is exactly where create–delete–recreate exploits historically appeared (refund capping in EIP-3529 was a partial patch, motivated in part by gas token abuse, i.e., a real-world splitting/deletion cycle).
- **State rent** proposals (EIP-1682 and predecessors) charge per-account-per-time rent; they were never deployed, largely due to access-preservation and eviction-complexity concerns — a known dead end in naive form.
- Economics literature on durable-goods externalities and deposit–refund systems (Bohm; Dinan on recycled-content tradeable credits) gives the closest analogue: charge a deposit at creation, refund at deletion. No blockchain formalization with manipulation-resistance proofs exists (verify).

## What would count as solving it?

1. **Formal mechanism + proof:** Define the resource model (state as a function of transaction history), and exhibit a pricing/refund rule with a proof that (a) no splitting or deletion-recreation strategy yields positive profit net of fees, and (b) long-run state size is bounded at target under myopic and adversarial users.
2. **Optimality extension:** Extend the ADM (2024) near-optimality theorem to a resource whose cost is non-separable/intertemporal, quantifying the welfare loss of any practical single-shot pricing rule relative to the clairvoyant optimum.
3. **Lower bound / impossibility (valuable negative result):** Prove that no transaction-local, history-independent price can simultaneously be manipulation-resistant, access-preserving, and budget-balanced — this would redirect effort to history-dependent or deposit-based schemes.
4. **Construction with experiment:** A concrete rule (e.g., deposit–refund with decay) validated in simulation against adversarial workload generators.

## How to start

1. Read Angeris–Diamandis–Moallemi (arXiv:2402.08661), especially the section on the resource-cost model and the "essentially optimal" competitive-ratio argument; identify exactly where separability is invoked — that is the lemma you must relax.
2. Write down the atomic adversary games: (i) split one large write into n small ones; (ii) write, delete, rewrite (gas-token style); (iii) hold state hostage (touch minimally to avoid eviction). Compute profit under candidate rules. This is cheap and immediately rules out naive schemes.
3. Model it as a deposit–refund ( externality) pricing problem: fee = deposit D at creation, refund r ≤ D at deletion, with the node-storage marginal cost as the externality benchmark. Check budget balance in steady state.
4. Main obstacle: state cost is a function of *aggregate history*, so any per-transaction posted price is either exploitable or requires global state-size-dependent prices, which reintroduces feedback-loop stability questions — connect to the control-theoretic analysis of EIP-1559 (e.g., the Leonardos–Reijsbergen–Tasiopoulos line, verify) for stability tools.

## Key references

- Guillaume Angeris, Theo Diamandis, Ciamac Moallemi. *Multidimensional Blockchain Fees are (Essentially) Optimal*. arXiv:2402.08661 (2024). https://arxiv.org/abs/2402.08661 — Proves multidimensional EIP-1559-style dynamic fees achieve near-optimal social welfare when block resource costs are convex and separable; the framework this problem must extend.
- Ethereum EIP-3529 (Berlin). https://eips.ethereum.org/EIPS/eip-3529 — Reduced gas refunds, explicitly curtailing gas-token usage, i.e., the deployed patch against a real deletion-cycle exploit.
- Ethereum EIP-1682 (state rent). https://eips.ethereum.org/EIPS/eip-1682 — An undeployed proposal for per-account per-time state rent; the canonical prior art whose access-preservation failures motivate this problem.
- Peter Bohm. *Deposit-Refund Systems: Theory and Applications to Environmental, Conservation, and Climate Policy* (2019, WSPC). — The environmental-economics theory of pricing durable externalities via upfront deposits with conditional refunds, the closest analogue outside blockchain. (verify edition/publisher)

## Warnings

- Do not assume state cost is separable: the whole difficulty is intertemporal/non-separable cost; ignoring this makes the problem trivially "solved" by ADM.
- "Access-preserving" is subjective — pin down a formal criterion (e.g., cold-access price within a constant factor of current mainnet) or referees will dispute it.
- Beware refund-arithmetics exploits: any refund r comparable to the deposit D invites write–delete–rewrite farming once combined with any fee-discount mechanism (base fee dips, blobs).
- History-dependent prices can be gamed by manipulating the price feed itself (self-referential feedback); prove stability, don't assume it.
- The census record is thin; the problem statement itself is the 2024–2025 research frontier, so expect to formalize the model from scratch rather than inherit one.

---
*`BCGT-000288` · flagged for review*

## Acceptance guidance

1. **Formal mechanism + proof:** Define the resource model (state as a function of transaction history), and exhibit a pricing/refund rule with a proof that (a) no splitting or deletion-recreation strategy yields positive profit net of fees, and (b) long-run state size is bounded at target under myopic and adversarial users.
2. **Optimality extension:** Extend the ADM (2024) near-optimality theorem to a resource whose cost is non-separable/intertemporal, quantifying the welfare loss of any practical single-shot pricing rule relative to the clairvoyant optimum.
3. **Lower bound / impossibility (valuable negative result):** Prove that no transaction-local, history-independent price can simultaneously be manipulation-resistant, access-preserving, and budget-balanced — this would redirect effort to history-dependent or deposit-based schemes.
4. **Construction with experiment:** A concrete rule (e.g., deposit–refund with decay) validated in simulation against adversarial workload generators.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/5dcee270-8c55-4c61-b7fc-77471343f9b6?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000288.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000288)
