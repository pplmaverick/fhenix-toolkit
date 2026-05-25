# Operator pattern — replacing ERC-20 allowance for ciphertexts

## What

ERC-20's `approve` / `transferFrom` cannot be replicated for encrypted balances. The allowance check (`require(allowance[from][spender] >= amount)`) needs a plaintext comparison, but both sides are ciphertexts and `require` doesn't work on `ebool`.

The replacement is the **operator pattern**: an account designates a per-pair operator (often a contract) with a time-bounded grant, and the encrypted-token contract checks `isOperator(account, operator)` (a plaintext bool) before pulling funds.

## The shape

```
contract FHERC20 {
    mapping(address => mapping(address => uint64)) public operatorUntil;
    // operatorUntil[owner][operator] = unix timestamp after which the grant expires

    function setOperator(address operator, uint64 until) external {
        operatorUntil[msg.sender][operator] = until;
        emit OperatorSet(msg.sender, operator, until);
    }

    function isOperator(address owner, address operator) public view returns (bool) {
        return operatorUntil[owner][operator] >= block.timestamp;
    }

    function confidentialTransferFrom(address from, address to, euint64 amount) external {
        require(msg.sender == from || isOperator(from, msg.sender), "not allowed");
        // ... encrypted transfer logic, FHE.select for balance bounding, etc.
    }
}
```

Note: the *fact* of who's an operator is public (it's a plaintext mapping). The *amounts* stay private.

## The rule

- **Replace every `approve(spender, amount)` call site** with `setOperator(spender, until)`.
- **Replace every `transferFrom(from, to, amount)` callsite check** with `require(isOperator(from, msg.sender))`.
- **Always include an `until` timestamp.** No "unlimited" approvals — encrypted-token operator grants should expire.
- **Frontends must issue two txs to swap via an integrator**: `setOperator` first, then the swap function. Document this clearly.

## Canonical example

- **RFQ — operator-gated pull from maker AND buyer.**
  https://github.com/FhenixProtocol/rfq-demo
  → `contracts/rfq.sol`. The `executeSwap(...)` function reverts with `MissingMakerOperator` or `MissingBuyerOperator` if either side hasn't set the RFQ contract as their operator. Two `setOperator` calls happen before the swap.

## Gotchas

- **`approve` / `allowance` from the OpenZeppelin ERC-20 surface won't compile** on a confidential token. Don't try to keep them as no-ops — remove them.
- **No partial allowances.** Operator grants are binary (on or off until expiry). If you need "allow up to N", you must enforce N via `FHE.select` inside the transfer logic.
- **Operator expiry is plaintext.** That's intentional — discoverable expiry is part of the safety story — but means the *timing* of revocation is observable.
- **No `permit` (EIP-2612) analogue ships out of the box.** Some Fhenix repos build their own (see `rfq-demo/contracts/cofhe/FHERC20Permit.sol`); reuse rather than reinvent.
