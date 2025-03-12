# EigenLayer Audit Competition Out-of-Scope Findings Report

**Date**: March 12, 2025  
**Prepared by**: BZ  
**X**: @pythoninsa  
**discord**: @_bzea.

**Purpose**: This report consolidates all findings identified as out of scope during the EigenLayer audit competition, categorized into Known Issues, Previous Audit Report Findings, and Lightchaser Report Findings, as per the provided documentation.

---

## 1. Known Issues

These are issues explicitly identified as "Known Issue" or designated out of scope due to being handled in a separate release, part of test files/tools/scripts, or documented risks/edge cases.

- **General Scope Statement**:  
  - "Out of scope findings during the eigen layer audit competition"

- **Pectra Compatibility Known Issues (v1.3.0)**:  
  - "The following known issues with v1.3.0 Pectra compatibility are out of scope, as we will be handling them in a separate release:"
    - **Proof sizes are changing**: "We do not support the new Pectra proofs in this scope."
    - **0x02 Withdrawal Credentials**: "We do not support verifying validators with 0x02 withdrawal credentials, though it is possible for existing verified validators to be the target of consolidation outside of a pod, moving them to 0x02 credentials."
    - **Validator Consolidation or Execution Layer Triggered Withdrawals**: "We do not support validator consolidation or execution layer triggered withdrawals."

- **Additional Known Issues**:  
  - **Test Files**: Out of scope.  
  - **Tools and Offchain Components**: Out of scope.  
  - **Anything in /scripts/ Folder**: Examples include upgrade scripts; out of scope.  
  - **Holesky Contracts**: "Our current Holesky contracts are on a slightly stale version of Slashing. Because of the current state of Holesky." Out of scope.  
  - **Risks/Edge Cases in Documentation**: "All risks/edge cases mentioned in the documentation are OOS (outta scope)."

---

## 2. Previous Audit Report Findings

This section compiles all findings from previous audit reports explicitly referenced in the input, spanning multiple reviews by ConsenSys Diligence, Sigma Prime, Cantina, Certora, and others. Each finding includes its ID (where provided), description, severity, status, and recommendations.

### ConsenSys Diligence Audit (March 2023)

- **Date**: March 22, 2023 - April 11, 2023  
- **Auditors**: Heiko Fisch, Dominik Muhs  
- **Commit Hash**: `0aa1dc7b65f3a7a7b718d6c8bd847612cbf2ae08`  
- **Files in Scope**: See Appendix 1 in original document (e.g., `StrategyManager.sol`, `EigenPod.sol`, etc.).

#### Findings

- **4.1 Potential Reentrancy Into Strategies**  
  - **Severity**: Medium  
  - **Description**: The `StrategyManager` contract’s deposit and withdrawal functions involve token transfers that could enable reentrancy if the token allows it. `ReentrancyGuardUpgradeable` protects `StrategyManager`, but `StrategyBase` lacks such protection. View functions like `sharesToUnderlyingView` may return inconsistent results during reentrancy due to state changes (e.g., shares burnt before token balance updates in `withdraw`).  
  - **Code Example**:  
    ```solidity
    function withdraw(address depositor, IERC20 token, uint256 amountShares) external {...}
    function sharesToUnderlyingView(uint256 amountShares) public view {...}
    ```
  - **Recommendation**: Use comprehensive reentrancy guards across all external/public functions, including view functions, and check `StrategyManager`’s reentrancy lock in `StrategyBase`. Split public functions into internal/external variants for clarity.  
  - **Status**: Not explicitly resolved in provided text.

- **4.2 StrategyBase – Inflation Attack Prevention Can Lead to Stuck Funds**  
  - **Severity**: Minor  
  - **Description**: `StrategyBase` enforces a minimum share threshold (10^9) to prevent inflation attacks, but this can trap funds up to 10^9 - 1 shares if total shares fall below this threshold.  
  - **Code Example**:  
    ```solidity
    require(updatedTotalShares >= MIN_NONZERO_TOTAL_SHARES, "...");
    ```
  - **Recommendation**: Consider internal accounting of underlying tokens or virtual shares to avoid stuck funds, though each has trade-offs (e.g., higher gas costs).  
  - **Status**: Not explicitly resolved.

- **4.3 StrategyWrapper – Functions Shouldn’t Be virtual**  
  - **Severity**: Minor (Out of Scope)  
  - **Description**: `StrategyWrapper` functions are marked `virtual`, contradicting its NatSpec that it’s not designed for inheritance.  
  - **Code Example**:  
    ```solidity
    contract StrategyWrapper is IStrategy {...} // Not designed to be inherited from
    ```
  - **Recommendation**: Remove `virtual` keyword if inheritance isn’t intended, or update documentation.  
  - **Status**: Explicitly noted as out of scope.

- **4.4 StrategyBase – Inheritance-Related Issues**  
  - **Severity**: Minor  
  - **Description**: Non-view function `underlyingToShares` in `StrategyBase` is forced to be `view` due to `IStrategy` interface, limiting overrides in derived contracts.  
  - **Code Example**:  
    ```solidity
    function underlyingToShares(uint256 amountUnderlying) external view returns (uint256);
    ```
  - **Recommendation**: Remove `view` modifier from `IStrategy` interface.  
  - **Status**: Not explicitly resolved.

- **4.12 Inconsistent Data Types for Block Numbers**  
  - **Severity**: Not specified  
  - **Description**: Block numbers use `uint32` and `uint64` inconsistently across contracts (e.g., `EigenPod.sol`, `StrategyManager.sol`).  
  - **Code Example**:  
    ```solidity
    uint32 creationBlockNumber; // EigenPod.sol
    uint64 oracleBlockNumber; // EigenPod.sol
    ```
  - **Recommendation**: Standardize to one data type (e.g., `uint64`) to avoid conversion errors.  
  - **Status**: Not explicitly resolved.

- **4.13 EigenPod – Stray nonReentrant Modifier**  
  - **Severity**: Not specified  
  - **Description**: `withdrawRestakedBeaconChainETH` in `EigenPod.sol` has a `nonReentrant` modifier, but no direct ETH transfer occurs (uses `DelayedWithdrawalRouter`), making it unnecessary.  
  - **Code Example**:  
    ```solidity
    function withdrawRestakedBeaconChainETH(address recipient, uint256 amountWei) external onlyEigenPodManager nonReentrant {...}
    ```
  - **Recommendation**: Remove `nonReentrant` modifier and related imports for clarity and gas savings.  
  - **Status**: Not explicitly resolved.

---

### Sigma Prime Audit (EigenLayer Detailed Findings)

#### Findings

- **EGN2-01 Middleware can Deny Withdrawals by Revoking Slashing Prior to Queueing Withdrawal**  
  - **Severity**: High (Impact: High, Likelihood: Medium)  
  - **Assets**: `Slasher.sol`, `StrategyManager.sol`  
  - **Description**: If middleware revokes slashing ability via `recordLastStakeUpdateAndRevokeSlashingAbility()` before a withdrawal is queued, `StrategyManager.canWithdraw()` fails, blocking withdrawals for stakers with one middleware.  
  - **Recommendation**: Validate only the `latestServeUntil` condition when a single middleware is revoked.  
  - **Status**: Resolved (commit `42bfa82c`).

- **EGN2-02 Domain Separator not Recalculated in Case of a Hard Fork**  
  - **Severity**: Low (Impact: Low, Likelihood: Low)  
  - **Assets**: `DelegationManager.sol`, `StrategyManager.sol`  
  - **Description**: `DOMAIN_SEPARATOR` is static post-initialization, risking signature replay across forked chains if `block.chainid` changes.  
  - **Recommendation**: Add `getDomainSeparator()` to dynamically recalculate based on current `block.chainid`.  
  - **Status**: Resolved (commit `714dbb61`).

- **EGN2-03 Delayed Withdrawals can be Created During Paused State**  
  - **Severity**: Informational  
  - **Asset**: `DelayedWithdrawalRouter.sol`  
  - **Description**: `createDelayedWithdrawal()` lacks a pause modifier, allowing withdrawals during a paused state, potentially bypassing delay periods.  
  - **Recommendation**: Add `onlyWhenNotPaused` modifier to `createDelayedWithdrawal()`.  
  - **Status**: Resolved (commit `721ced7e`).

- **EGN2-04 Funds can be Lost Due to SelfDestructed Staker Contract**  
  - **Severity**: Informational  
  - **Asset**: `StrategyManager.sol`  
  - **Description**: A staker contract using `SELFDESTRUCT` before `depositIntoStrategyWithSignature()` can deposit funds but cannot withdraw due to destroyed bytecode.  
  - **Recommendation**: Provide a backup withdrawal address or document avoidance of `SELFDESTRUCT`.  
  - **Status**: Closed (no fix planned, acknowledged as unaddressed).

- **EGN2-05 Novel Staking Risks Posed by Middleware**  
  - **Severity**: Informational  
  - **Asset**: `Slasher.sol`  
  - **Description**: Middleware introduces compounded risks across protocols, potentially making slashing profitable if rewards outweigh penalties.  
  - **Recommendation**: Enhance documentation on middleware risks and consider functionality for risk-based operator selection.  
  - **Status**: Resolved (documentation updates planned).

- **EGN2-06 Miscellaneous General Comments**  
  - **Severity**: Informational  
  - **Description**: General observations resolved with EGN2-05.  
  - **Status**: Resolved.

---

### Cantina Competition (April 5, 2024)

- **Date**: February 27 - March 18, 2024  

#### Findings

- **3.1.1 Beacon Chain Withdrawals That Occur at lastwithdrawaltimestamp Will Be Lost**  
  - **Severity**: High  
  - **Context**: `EigenPod.sol` (lines 119-126, 381-391, 566-583, 733-737)  
  - **Description**: ETH withdrawn at `mostRecentWithdrawalTimestamp` during `activateRestaking` is lost due to EIP-4895 execution timing, as `verifyAndProcessWithdrawals` requires `timestamp > mostRecentWithdrawalTimestamp`.  
  - **Proof of Concept**:  
    1. Withdrawal of 32 ETH at time `t`.  
    2. `activateRestaking` called at `t`, but ETH isn’t included in `address(this).balance`.  
    3. Post-`t`, ETH cannot be withdrawn due to timestamp check.  
  - **Recommendation**: Use `>= mostRecentWithdrawalTimestamp` in the modifier.  
  - **Status**: Not explicitly resolved.

- **3.2.1 External LST Compromission Leads to AVS Compromission Due to Lack of Operators' Staked Amount Cooldown**  
  - **Severity**: Medium  
  - **Context**: `DelegationManager.sol`, `EigenDAServiceManager.sol`, `RegistryCoordinator.sol`, `StrategyManager.sol`  
  - **Description**: A compromised LST allows an attacker to mint, deposit, and register as an operator within two blocks, compromising AVSs without cooldown protection.  
  - **Proof of Concept**: Mint LST, deposit via `StrategyManager.depositIntoStrategy()`, register via `RegistryCoordinator.registerOperator()`, and influence `EigenDAServiceManager.confirmBatch`.  
  - **Recommendation**: Rate-limit LST deposits in `StrategyBase.deposit` (e.g., patch with `rateLimitAmountPerBlock`).  
  - **Status**: Not explicitly resolved.

- **3.2.2 Paused Deregistering is Bypassed via registerOperatorWithChurn**  
  - **Severity**: Medium  
  - **Context**: Not provided  
  - **Description**: `deregisterOperator` is pausable, but `registerOperatorWithChurn` can deregister operators during a pause if conditions are met.  
  - **Code Example**:  
    ```solidity
    function deregisterOperator(bytes calldata quorumNumbers) external onlyWhenNotPaused(PAUSED_DEREGISTER_OPERATOR) {...}
    function registerOperatorWithChurn(...) external {... _deregisterOperator(...); ...}
    ```
  - **Recommendation**: Check pause status in `_deregisterOperator`.  
  - **Status**: Not explicitly resolved.

---

### Sigma Prime - Checkpoint Proofs

- **Commit**: `d148952`  
- **Scope**: `interfaces/IEigenPod.sol`, `interfaces/IEigenPodManager.sol`, `libraries/BeaconChainProofs.sol`, `pods/*`

#### Findings

- **EGN5-01 Proofs Based On beaconStateRoot Break After Electra Upgrade**  
  - **Severity**: Critical (Impact: High, Likelihood: High)  
  - **Assets**: `BeaconChainProofs.sol`, `EigenPod.sol`  
  - **Description**: Electra increases `BeaconState` tree height from 5 to 6, breaking proofs in `verifyValidatorFields()` and `verifyBalanceContainer()` using `BEACON_STATE_TREE_HEIGHT = 5`. Also enables a second pre-image attack for high `validatorIndex`.  
  - **Recommendation**: Adjust tree height post-Electra fork timestamp.  
  - **Status**: Closed ("Fix planned post-Electra finalization").

- **EGN5-02 Lack Of Support For Compounding Withdrawal Credentials After Electra Upgrade**  
  - **Severity**: Medium  
  - **Description**: Post-Electra compounding credentials not supported.  
  - **Status**: Closed.

- **EGN5-03 Validator Status Incorrectly Set To WITHDRAWN After Electra Upgrade**  
  - **Severity**: Medium  
  - **Description**: Validator status misclassification post-Electra.  
  - **Status**: Closed.

- **EGN5-04 Double-Counting ETH From Consolidations After Electra Upgrade**  
  - **Severity**: Informational  
  - **Description**: ETH from consolidations may be double-counted post-Electra.  
  - **Status**: Closed.

- **EGN5-05 Double-Counting Partial Withdrawals After Electra Upgrade**  
  - **Severity**: Informational  
  - **Description**: Partial withdrawals may be double-counted post-Electra.  
  - **Status**: Closed.

- **EGN5-06 Miscellaneous General Comments**  
  - **Severity**: Informational  
  - **Description**: General observations acknowledged.  
  - **Status**: Closed.

---

### Sigma Prime - Rewards v2

#### Findings

- **ELSC2-13 Hardcoded Default Operator Split Values Used In Reward Calculations**  
  - **Severity**: Informational  
  - **Asset**: `rewards/*`  
  - **Description**: Hardcoded 10% operator split in SQL queries mismatches on-chain changes, risking incorrect distributions.  
  - **Recommendation**: Use a state model with `default_operator_split_snapshots` table.  
  - **Status**: Resolved (commit `2941257`).

---

### Sigma Prime - Rewards Coordinator

#### Findings

- **EG4-01 Repeated Identical Range Rewards Will Be Allowed In Separate Transactions**  
  - **Severity**: Medium (Impact: Medium, Likelihood: Medium)  
  - **Asset**: `RewardsCoordinator.sol`  
  - **Description**: Unique `submissionNonce` ensures distinct reward hashes, allowing duplicate range rewards to be processed and paid out.  
  - **Recommendation**: Exclude nonces from hash calculations.  
  - **Status**: Open.

- **EG4-02 Lack Of Access Control Checks On initialize()**  
  - **Severity**: Low (Impact: Medium, Likelihood: Low)  
  - **Asset**: `RewardsCoordinator.sol`  
  - **Description**: No access control on `initialize()`, risking front-running during deployment.  
  - **Recommendation**: Add access controls or deploy/initialize in one transaction.  
  - **Status**: Open.

- **EG4-03 Rounding In Calculations Leaves Tokens Unassigned**  
  - **Severity**: Low  
  - **Asset**: `RewardsCoordinator.sol`  
  - **Description**: Rounding errors leave tokens unallocated.  
  - **Status**: Open.

- **EG4-04 Unnecessary Same Day Registration Calculation**  
  - **Severity**: Informational  
  - **Asset**: EigenLayer Rewards Calculation  
  - **Description**: Same-day registrations filtered unnecessarily; `start_time < end_time` could simplify.  
  - **Recommendation**: Simplify condition to `start_time < end_time`.  
  - **Status**: Open.  

- **EG4-05 Miscellaneous General Comments**  
  - **Severity**: Informational  
  - **Asset**: `RewardsCoordinator.sol`  
  - **Description**:  
    1. `beaconChainETHStrategy` not in block capitals.  
    2. Gas: Check constants before external calls (line 341).  
    3. Gas: Loops in `createAVSRewardsSubmission`/`createRewardsForAllSubmission` risk high costs/reverts.  
  - **Recommendation**: Address naming, optimize gas usage.  
  - **Status**: Open.

---

### Sigma Prime - EIGEN Rewards

#### Findings

- **EGN7-05 Inactive EIGEN Supply Stuck After Token Upgrade**  
  - **Severity**: Informational  
  - **Asset**: `Eigen.sol`  
  - **Description**: Pre-upgrade unwrapped EIGEN tokens stuck in `Eigen` contract post-upgrade, excluded from circulating supply.  
  - **Code Example**:  
    ```solidity
    function unwrap(uint256 amount) external {...}
    ```
  - **Recommendation**: Burn stuck tokens post-upgrade.  
  - **Status**: Closed ("No real consequences").

- **EGN7-06 Miscellaneous General Comments**  
  - **Severity**: Informational  
  - **Assets**: `TokenHopper.sol`, `RewardAllStakersActionGenerator.sol`  
  - **Description**:  
    1. **Missing Timestamp Validation**: Underflow risks in `generateHopperActions()` and `pressButton()` if called before start timestamps.  
    2. **Code Readability**: Complex conditions could be split for clarity (e.g., `_canPress()`, `pressButton()`).  
  - **Recommendation**: Add `require()` checks for timestamps, split conditions.  
  - **Status**: Closed (fixes in commits `a839ab6`, partial `8709beb`).

---

### Sigma Prime - Slashing Review

#### Findings

- **EGSL-18 Unsafe Casting In OperatorSetLib.decode()**  
  - **Severity**: Informational  
  - **Asset**: `OperatorSetLib.sol`  
  - **Description**: `decode()` casts `uint256` to `uint32` unsafely, but only used with encoded keys, mitigating risk.  
  - **Code Example**:  
    ```solidity
    function decode(bytes32 _key) internal pure returns (OperatorSet memory) {...}
    ```
  - **Recommendation**: Use safe casting with revert checks, remove unnecessary `& type(uint96).max`.  
  - **Status**: Closed (no fix, no attack vector).

- **EGSL-19 Miscellaneous General Comments**  
  - **Severity**: Informational  
  - **Assets**: `AllocationManager.sol`, `DelegationManager.sol`, `EigenPodManager.sol`, `EigenPod.sol`  
  - **Description**:  
    1. Redundant `StrategyAddedToOperatorSet` events in `createOperatorSets()`.  
    2. Gas: Declare arrays outside loops in `_undelegate()`.  
    3. Missing return value check in `_removeSharesAndQueueWithdrawal()`.  
    4. Inconsistent NatSpec on Gwei requirements (post-slashing).  
    5. `modifyAllocations()` allows empty `params`.  
  - **Recommendation**: Prevent redundant events, optimize gas, add checks, update NatSpec, require non-empty `params`.  
  - **Status**: Closed.

---

### Certora Formal Verification

- **Modules**: Merkle, Endian, BeaconChainProofs, Delegation Manager, EigenPod Manager

#### Merkle Module
- **P-21 merkleizeSha256 doesn't collide**:  
  - **Status**: Violated (`merkleizeSha256IsInjective`), Verified (`merkleizeSha256IsInjective_onSameLengths`).

#### Endian Module
- **P-23 fromLittleEndianUint64 works correctly**:  
  - **Status**: Violated (`fromLittleEndianUint64_correctness`), Verified (`transformationsAreInverse1`, `transformationsAreInverse2`).

#### BeaconChainProofs Module
- **P-24 verifyValidatorBalance works correctly**:  
  - **Status**: Violated (`verifyValidatorBalance_balanceRootUnique`).

#### Delegation Manager Module
- **P-01 CumulativeScaledShares is monotonically increasing**: Verified.  
- **P-02 CumulativeScaledSharesHistory Keys are valid**: Verified.  
- **P-03 Operator Cannot Deregister**: Verified.  
- **P-04 Queued Withdrawal Registration Consistency Invariant**: Verified.  
- **P-05 Authorized Methods for Deposit Scaling Factor Modification Invariant**: Verified.  
- **P-06 Operators delegate to themselves**: Verified.

#### EigenPod Manager Module
- **P-01 Pod owner deposited shares remain positive**: Verified.  
- **P-02 BeaconChainSlashingFactor is monotonically decreasing**: Verified.  
- **P-03 Pod Address immutability**: Verified.  
- **P-04 Remove Deposit Shares Integrity**: Verified.  
- **P-05 Add Shares Integrity**: Violated.  
- **P-06 Integrity Of Withdraw Shares As Tokens**: Verified.  
- **P-07 Add-Then-Remove Shares Neutrality**: Verified.  
- **P-08 Remove-Then-Add Shares Neutrality**: Verified.  
- **P-17 podOwner shares are whole Gwei amount**: Verified.  
- **P-18 Limitation on negative shares**: Verified.  
- **P-19 addShares and removeShares are inverse**: Verified.  
- **P-20 add and removeShares revert when the podOwner doesn’t have a pod**: Violated.

#### Allocation Manager Module
- **P-05 Pending Diffs valid state transitions**: Verified.  
- **P-06 Deallocations effect blocks are in ascending order**: Violated.  
- **P-07 DeallocationQueue allocations uniqueness**: Violated.  
- **P-08 Deallocation Queue Effect Block Timing Bound Invariant**: Violated.

- **Status**:  assumed out of scope as prior formal verification.

---

## 3. Lightchaser Report Findings

- **Reference**: [https://gist.github.com/ChaseTheLight01/a72a2fd43d2aacf9a11aebe93811ce9a]

# LightChaser-V3 Findings Report

**Generated for**: Cantina: EigenLayer  
**Generated on**: 2025-03-08  
**Total Findings**: 229  
- **Medium Findings**: 5  
- **Low Findings**: 70  
- **Gas Findings**: 72  
- **NonCritical Findings**: 82  

---

## Summary of Findings

### Medium Findings

| Number      | Details                                                                                              | Instances |
|-------------|------------------------------------------------------------------------------------------------------|-----------|
| [Medium-1]  | Permanent DoS due to non-shrinking array usage in an unbounded loop                                  | 4         |
| [Medium-2]  | Privileged functions can create points of failure                                                    | 42        |
| [Medium-3]  | Inconsistent type hash definition which doesn't match the struct that it's supposed to represent which may result in failed signature verifications | 1         |
| [Medium-4]  | Withdrawal mechanism can be DoS'd with dust withdrawals                                              | 2         |
| [Medium-5]  | Function uses `gasLeft()` or `gas()` as a parameter of an external call without first checking if `gasLeft()` is sufficient when considering the 63/64 rule (EIP-150) | 1         |

---

### Low Findings

| Number      | Details                                                                                              | Instances |
|-------------|------------------------------------------------------------------------------------------------------|-----------|
| [Low-1]     | Potential division by zero should have zero checks in place                                          | 4         |
| [Low-2]     | Missing checks for `address(0x0)` when updating address state variables                             | 12        |
| [Low-3]     | Solidity version 0.8.23 won't work on all chains due to `MCOPY`                                      | 3         |
| [Low-4]     | Function call without checking or using its return values                                            | 2         |
| [Low-5]     | Contracts do not use their OZ upgradable counterparts                                                | 8         |
| [Low-6]     | Some tokens may revert when zero value transfers are made                                            | 12        |
| [Low-7]     | Using `block.number` for time comparisons                                                            | 8         |
| [Low-8]     | Int casting `block.timestamp` can reduce the lifespan of a contract                                  | 1         |
| [Low-9]     | Low level calls in Solidity versions preceding 0.8.14 can result in an optimizer bug                 | 7         |
| [Low-10]    | The call `abi.encodeWithSelector` is not type safe                                                   | 2         |
| [Low-11]    | Upgradable contracts should have a `__gap` variable                                                  | 11        |
| [Low-12]    | Using `>` when declaring Solidity version without specifying an upper bound can cause future vulnerabilities | 1         |
| [Low-13]    | Contracts are vulnerable to fee-on-transfer accounting-related issues                                | 7         |
| [Low-14]    | Use `SafeCast` to safely downcast variables                                                          | 15        |
| [Low-15]    | Upgradable contracts should have an initialization function                                          | 1         |
| [Low-16]    | The `nonReentrant` modifier should be first in a function declaration                                | 29        |
| [Low-17]    | No limits when setting min/max amounts                                                               | 1         |
| [Low-18]    | Initializer function can be front-run                                                                | 11        |
| [Low-19]    | The function `decimals()` is not part of the ERC20 standard                                          | 1         |
| [Low-20]    | Minting to the zero address should be avoided                                                        | 1         |
| [Low-21]    | Contract inherits `Pausable` but does not expose pausing/unpausing functionality                     | 1         |
| [Low-22]    | Uses of EIP712 does not include a salt                                                               | 1         |
| [Low-23]    | Loss of precision                                                                                    | 5         |
| [Low-24]    | Missing zero address check in constructor                                                            | 3         |
| [Low-25]    | `Ownable` is inherited but not used                                                                  | 4         |
| [Low-26]    | Return values of `transfer()/transferFrom()` not checked                                             | 2         |
| [Low-27]    | Use of `onlyOwner` functions can be lost                                                             | 5         |
| [Low-28]    | Off-by-one timestamp error                                                                           | 5         |
| [Low-29]    | Contract can be bricked by the use of both `Ownable` and `Pausable` in the same contract             | 14        |
| [Low-30]    | `approve()/safeApprove()` may revert if the current approval is not zero                             | 1         |
| [Low-31]    | Remaining ETH may not be refunded to users                                                           | 1         |
| [Low-32]    | Constant decimal values                                                                              | 2         |
| [Low-33]    | Arrays can grow in size without a way to shrink them                                                 | 1         |
| [Low-34]    | Absence of Reentrancy Guard with OpenZeppelin's `sendValue` function usage                           | 1         |
| [Low-35]    | Sending tokens in a `for` loop                                                                       | 17        |
| [Low-36]    | Large approvals such as `type(uint256).max` may not work with some tokens                            | 1         |
| [Low-37]    | Revert on transfer to the zero address                                                               | 3         |
| [Low-38]    | Return values not checked for `approve()`                                                            | 1         |
| [Low-39]    | Missing zero address check in initializer                                                            | 7         |
| [Low-40]    | Critical functions should have a timelock                                                            | 11        |
| [Low-41]    | Unbounded loop may run out of gas                                                                    | 34        |
| [Low-42]    | Consider implementing two-step procedure for updating protocol addresses                             | 10        |
| [Low-43]    | Don't assume specific ETH balance                                                                    | 1         |
| [Low-44]    | Avoid mutating function parameters                                                                   | 2         |
| [Low-45]    | Prefer skip over revert model in iteration                                                           | 17        |
| [Low-46]    | Numbers downcast to addresses may result in collisions                                               | 2         |
| [Low-47]    | Inconsistent expiry logic using block global values                                                  | 1         |
| [Low-48]    | Constructors missing validation                                                                      | 11        |
| [Low-49]    | Inconsistent use of `_msgSender()` and `msg.sender` in contract                                      | 1         |
| [Low-50]    | Division in comparison                                                                               | 2         |
| [Low-51]    | State variables not capped at reasonable values                                                      | 5         |
| [Low-52]    | Use `forceApprove` in place of `approve`                                                             | 1         |
| [Low-53]    | Unsafe `uint` to `int` conversion                                                                    | 2         |
| [Low-54]    | Unsafe use of `transfer()/transferFrom()` with IERC20                                                | 3         |
| [Low-55]    | Large transfers may not work with some ERC20 tokens                                                  | 12        |
| [Low-56]    | Functions calling contracts/addresses with transfer hooks are missing reentrancy guards              | 12        |
| [Low-57]    | Solidity version 0.8.20 won't work on all chains due to `PUSH0`                                      | 3         |
| [Low-58]    | Using `block.number` is not fully L2 compatible                                                      | 17        |
| [Low-59]    | Missing events in functions that are either setters, privileged, or voting-related                   | 42        |
| [Low-60]    | Unsafe use of `transfer()/transferFrom()` with IERC20 (duplicate of Low-54)                          | 2         |
| [Low-61]    | Avoid floating pragma in non-interface/library files                                                 | 29        |
| [Low-62]    | Address function parameter is compared against a state variable; can be bypassed for proxy tokens    | 1         |
| [Low-63]    | Upgradeable contract uses `SafeERC20` rather than `SafeERC20Upgradeable`, not upgrade-safe due to `Address.sol` delegatecall | 3         |
| [Low-64]    | Read-only reentrancy risk detected                                                                   | 36        |
| [Low-65]    | Common tokens like WETH9 work differently on chains like Blast, not accounted for in transfer calls  | 1         |
| [Low-66]    | Project contains upgradeable base contracts with `__gap` variables, but some lack them               | 10        |
| [Low-67]    | Project contains upgradeable base contracts with `__gap`, but some non-upgradeable bases lack them   | 10        |
| [Low-68]    | Contract initialization function can fail due to out-of-gas error on certain chains due to iteration | 1         |
| [Low-69]    | Transfers in iteration do not compare `amounts[i]` against zero, risking revert                      | 9         |
| [Low-70]    | Consider an uptime feed on L2 deployments to prevent issues caused by downtime                       | 29        |

---

### NonCritical Findings

| Number            | Details                                                                                              | Instances |
|-------------------|------------------------------------------------------------------------------------------------------|-----------|
| [NonCritical-1]   | Addresses shouldn't be hard-coded                                                                    | 1         |
| [NonCritical-2]   | Pure function is not defined as such in interface                                                    | 1         |
| [NonCritical-3]   | Consider using time variables when defining time-related variables                                   | 2         |
| [NonCritical-4]   | Contracts with multiple `onlyXYZ` modifiers where XYZ is a role can introduce complexities           | 5         |
| [NonCritical-5]   | Local variable shadowing                                                                             | 2         |
| [NonCritical-6]   | Using `abi.encodePacked` can result in hash collision when used in hashing functions                 | 4         |
| [NonCritical-7]   | Greater-than comparisons made on state `uint`s that can be set to max                                | 1         |
| [NonCritical-8]   | Floating pragma should be avoided                                                                    | 4         |
| [NonCritical-9]   | Events regarding state variable changes should emit the previous state variable value                | 2         |
| [NonCritical-10]  | In functions accepting an address parameter, there should be a zero address check                   | 128       |
| [NonCritical-11]  | Enum values should be used in place of constant array indexes                                        | 8         |
| [NonCritical-12]  | Default int values are manually set                                                                  | 49        |
| [NonCritical-13]  | `Ownable2Step` should be used in place of `Ownable`                                                  | 9         |
| [NonCritical-14]  | Revert statements within external/public functions can be used to perform DoS attacks                | 1         |
| [NonCritical-15]  | Functions that are private or internal should have a preceding `_` in their name                     | 46        |
| [NonCritical-16]  | Private and internal state variables should have a preceding `_` unless they are constants           | 3         |
| [NonCritical-17]  | Contract lines should not be longer than 120 characters for readability                              | 299       |
| [NonCritical-18]  | Avoid updating storage when the value hasn't changed                                                 | 11        |
| [NonCritical-19]  | Specific imports should be used where possible so only used code is imported                         | 78        |
| [NonCritical-20]  | Old Solidity version                                                                                 | 3         |
| [NonCritical-21]  | Not all event definitions are utilizing indexed variables                                            | 6         |
| [NonCritical-22]  | Explicitly define visibility of state variables to prevent misconceptions                            | 1         |
| [NonCritical-23]  | Contracts should have all public/external functions exposed by interfaces                            | 177       |
| [NonCritical-24]  | `uint/int` variables should have the bit size defined explicitly                                     | 80        |
| [NonCritical-25]  | Functions within contracts are not ordered according to the Solidity style guide                    | 12        |
| [NonCritical-26]  | Double type casts create complexity within the code                                                  | 10        |
| [NonCritical-27]  | Emits without `msg.sender` parameter                                                                 | 3         |
| [NonCritical-28]  | Functions with array parameters should have length checks in place                                   | 7         |
| [NonCritical-29]  | All interfaces used within a project should be imported                                              | 2         |
| [NonCritical-30]  | A function defining named returns in its declaration doesn't need to use `return`                   | 7         |
| [NonCritical-31]  | Unused state variables present                                                                       | 43        |
| [NonCritical-32]  | Unused mappings present                                                                              | 7         |
| [NonCritical-33]  | Unused modifiers present                                                                             | 1         |
| [NonCritical-34]  | Constants should be on the left side of the comparison                                               | 29        |
| [NonCritical-35]  | Defined named returns not used within function                                                       | 2         |
| [NonCritical-36]  | Initialize functions do not emit an event                                                            | 9         |
| [NonCritical-37]  | Both immutable and constant state variables should be `CONSTANT_CASE`                                | 12        |
| [NonCritical-38]  | Using zero as a parameter                                                                            | 5         |
| [NonCritical-39]  | Use of non-named numeric constants                                                                   | 33        |
| [NonCritical-40]  | Redundant `else` statement                                                                           | 4         |
| [NonCritical-41]  | Use `immutable` not `constant` for `keccak` state variables                                          | 5         |
| [NonCritical-42]  | Event emit should emit a parameter                                                                   | 2         |
| [NonCritical-43]  | Unused structs present                                                                               | 5         |
| [NonCritical-44]  | Empty bytes check is missing                                                                         | 34        |
| [NonCritical-45]  | Use `max` instead of `0xfff...`                                                                      | 2         |
| [NonCritical-46]  | Return `bool` not explicit                                                                           | 1         |
| [NonCritical-47]  | Assembly block creates dirty bits                                                                    | 4         |
| [NonCritical-48]  | Cyclomatic complexity in functions                                                                   | 5         |
| [NonCritical-49]  | Incorrect withdraw declaration                                                                       | 1         |
| [NonCritical-50]  | Consider implementing EIP-5267 to securely describe EIP-712 domains being used                       | 1         |
| [NonCritical-51]  | Ensure `block.timestamp` is only used in long time intervals                                         | 1         |
| [NonCritical-52]  | An event should be emitted if a non-immutable state variable is set in a constructor                 | 2         |
| [NonCritical-53]  | Non-constant/immutable state variables are missing a setter post-deployment                          | 12        |
| [NonCritical-54]  | `uint` casted addresses used in conditional checks can be bypassed                                   | 1         |
| [NonCritical-55]  | Address collision possible due to upcast                                                             | 1         |
| [NonCritical-56]  | Pure function storage pointer bug                                                                    | 1         |
| [NonCritical-57]  | Nonce variables should be `uint256`                                                                  | 2         |
| [NonCritical-58]  | Floating pragma defined inconsistently                                                               | 4         |
| [NonCritical-59]  | Use `using` keyword when using specific imports rather than calling the specific import directly     | 20        |
| [NonCritical-60]  | Inconsistent checks of address params against `address(0)`                                           | 2         |
| [NonCritical-61]  | Avoid declaring variables with the names of defined functions within the project                    | 38        |
| [NonCritical-62]  | Simplify complex `require` statements                                                                | 2         |
| [NonCritical-63]  | Constructors should emit an event                                                                    | 20        |
| [NonCritical-64]  | Contract and abstract files should have a fixed compiler version                                     | 31        |
| [NonCritical-65]  | Function call in event emit                                                                          | 1         |
| [NonCritical-66]  | `int/uint` passed into `abi.encodePacked` without casting to a string                                | 1         |
| [NonCritical-67]  | `for` loop iterates on arrays not indexed                                                            | 27        |
| [NonCritical-68]  | Constructor with array/string/bytes parameters has no empty array checks                             | 12        |
| [NonCritical-69]  | Avoid external calls in modifiers                                                                    | 1         |
| [NonCritical-70]  | Events should have parameters                                                                        | 2         |
| [NonCritical-71]  | Errors should have parameters                                                                        | 86        |
| [NonCritical-72]  | Avoid arithmetic directly within array indices                                                       | 3         |
| [NonCritical-73]  | Return values not checked for OZ `EnumerableSet` add/remove functions                                | 3         |
| [NonCritical-74]  | Memory-safe annotation missing                                                                       | 1         |
| [NonCritical-75]  | Constant state variables defined more than once                                                      | 3         |
| [NonCritical-76]  | Modifier checks `msg.sender` against two addresses; consider using multiple modifiers for more control | 2         |
| [NonCritical-77]  | Unnecessary struct attribute prefix                                                                  | 14        |
| [NonCritical-78]  | ERC777 tokens can introduce reentrancy risks                                                         | 2         |
| [NonCritical-79]  | Avoid assembly in `view` or `pure` functions                                                         | 27        |
| [NonCritical-80]  | It is best practice to initialize a local variable when defined                                      | 15        |
| [NonCritical-81]  | Contract calls `disableInitializers` in its constructor but also contains an initialization function with `initializer` modifier | 34        |
| [NonCritical-82]  | Calls are made against a pausable contract without checking if the contract is paused                | 44        |

---

### Gas Findings

| Number      | Details                                                                                              | Instances | Gas       |
|-------------|------------------------------------------------------------------------------------------------------|-----------|-----------|
| [Gas-1]     | Mappings are cheaper than arrays for state variable iteration                                        | 4         | 16000     |
| [Gas-2]     | Avoid updating storage when the value hasn't changed                                                 | 3         | 7200      |
| [Gas-3]     | `x + y` is more efficient than using `+=` for state variables (likewise for `-=`)                    | 3         | 45        |
| [Gas-4]     | There is a 32-byte length threshold for error strings; longer strings consume more gas               | 13        | 2366      |
| [Gas-5]     | Public functions not used internally can be marked as `external` to save gas                         | 11        | 0.0       |
| [Gas-6]     | `calldata` should be used in place of `memory` function parameters when not mutated                  | 10        | 1300      |
| [Gas-7]     | Nested `for` loops should be avoided due to high gas costs resulting from O^2 time complexity        | 7         | 0.0       |
| [Gas-8]     | Usage of smaller `uint/int` types causes overhead                                                    | 287       | 4530295   |
| [Gas-9]     | Use `!= 0` instead of `> 0`                                                                          | 23        | 1587      |
| [Gas-10]    | Integer increments by one can be unchecked to save on gas fees                                       | 5         | 3000      |
| [Gas-11]    | Default `bool` values are manually reset                                                             | 1         | 0.0       |
| [Gas-12]    | Default `int` values are manually reset                                                              | 7         | 0.0       |
| [Gas-13]    | Function calls within `for` loops                                                                    | 73        | 0.0       |
| [Gas-14]    | `for` loops in public or external functions should be avoided due to high gas costs and possible DoS | 24        | 0.0       |
| [Gas-15]    | Mappings used within a function more than once should be cached to save gas                          | 3         | 900       |
| [Gas-16]    | Use assembly to check for the zero address                                                           | 19        | 0.0       |
| [Gas-17]    | Some error strings are not descriptive                                                               | 90        | 0.0       |
| [Gas-18]    | Divisions which do not divide by `-X` cannot overflow or underflow, so can be unchecked              | 5         | 0.0       |
| [Gas-19]    | Can transfer 0                                                                                       | 1         | 0.0       |
| [Gas-20]    | Redundant state variable getters                                                                     | 1         | 0.0       |
| [Gas-21]    | State variables not modified within functions should be set as `constant` or `immutable`             | 11        | 0.0       |
| [Gas-22]    | Divisions of powers of 2 can be replaced by a right shift operation to save gas                      | 4         | 0.0       |
| [Gas-23]    | Multiplications of powers of 2 can be replaced by a left shift operation to save gas                 | 1         | 0.0       |
| [Gas-24]    | Don't use `_msgSender()` if not supporting EIP-2771                                                  | 1         | 16        |
| [Gas-25]    | Private functions used once can be inlined                                                           | 1         | 0.0       |
| [Gas-26]    | Consider using OZ `EnumerateSet` in place of nested mappings                                         | 22        | 484000    |
| [Gas-27]    | Use `selfBalance()` in place of `address(this).balance`                                              | 1         | 800       |
| [Gas-28]    | Use assembly to emit events                                                                          | 96        | 350208    |
| [Gas-29]    | Use Solady library where possible to save gas                                                        | 5         | 25000     |
| [Gas-30]    | Use assembly in place of `abi.decode` to extract calldata values more efficiently                    | 1         | 0.0       |
| [Gas-31]    | Struct variables can be packed into fewer storage slots by truncating timestamp bytes                | 2         | 10000     |
| [Gas-32]    | Using `private` rather than `public` for constants and immutables saves gas                          | 7         | 0.0       |
| [Gas-33]    | Mark functions that revert for normal users as `payable`                                             | 41        | 42025     |
| [Gas-34]    | Lack of `unchecked` in loops                                                                         | 117       | 891540    |
| [Gas-35]    | Where a value is casted more than once, consider caching the result to save gas                      | 11        | 0.0       |
| [Gas-36]    | Assembly `let` var only used once                                                                    | 4         | 0.0       |
| [Gas-37]    | Assembly `let` var never used                                                                        | 1         | 0.0       |
| [Gas-38]    | Use assembly to validate `msg.sender`                                                                | 16        | 0.0       |
| [Gas-39]    | Simple checks for zero `uint` can be done using assembly to save gas                                 | 17        | 1734      |
| [Gas-40]    | Use `unchecked` for divisions on constant or immutable values                                        | 1         | 0.0       |
| [Gas-41]    | Using nested `if` to save gas                                                                        | 17        | 1734      |
| [Gas-42]    | Optimize storage with byte truncation for time-related state variables                               | 5         | 50000     |
| [Gas-43]    | Using `delete` instead of setting mapping to 0 saves gas                                             | 1         | 5         |
| [Gas-44]    | Stack variable costs less than state variables while used in emitting event                          | 12        | 1296      |
| [Gas-45]    | Stack variable costs less than structs while used in emitting event                                  | 2         | 36        |
| [Gas-46]    | Caching global variables is more expensive than using the actual variable                            | 1         | 12        |
| [Gas-47]    | Remove unused modifiers                                                                              | 1         | 0.0       |
| [Gas-48]    | Inline modifiers used only once                                                                      | 3         | 0.0       |
| [Gas-49]    | Use `s.x = s.x + y` instead of `s.x += y` for state variable structs                                 | 1         | 0.0       |
| [Gas-50]    | Use `s.x = s.x + y` instead of `s.x += y` for memory structs (same for `-=` etc.)                     | 3         | 900       |
| [Gas-51]    | Solidity versions 0.8.19 and above are more gas efficient                                            | 3         | 9000      |
| [Gas-52]    | Calling `.length` in a `for` loop wastes gas                                                         | 95        | 912285    |
| [Gas-53]    | Internal functions only used once can be inlined to save gas                                         | 36        | 38880     |
| [Gas-54]    | Constructors can be marked as `payable` to save deployment gas                                       | 22        | 0.0       |
| [Gas-55]    | Merge events to save gas                                                                             | 4         | 6000      |
| [Gas-56]    | Assigning to structs can be more efficient                                                           | 18        | 42120     |
| [Gas-57]    | An inefficient way of checking if an integer is even is being used (`X % 2 == 0`)                    | 1         | 0.0       |
| [Gas-58]    | Internal functions never used can be removed                                                         | 13        | 0.0       |
| [Gas-59]    | Only emit event in setter function if the state variable was changed                                 | 26        | 0.0       |
| [Gas-60]    | It is a waste of gas to emit variable literals                                                       | 1         | 8         |
| [Gas-61]    | Short circuit before external calls                                                                  | 3         | 0.0       |
| [Gas-62]    | Use OZ `Array.unsafeAccess()` to avoid repeated array length checks                                  | 111       | 25874100  |
| [Gas-63]    | State variable read in a loop                                                                        | 8         | 370816    |
| [Gas-64]    | Use `uint256(1)/uint256(2)` instead of `true/false` to save gas for changes                          | 18        | 5508000   |
| [Gas-65]    | Avoid emitting events in loops                                                                       | 70        | 1942500   |
| [Gas-66]    | Call `msg.XYZ` directly rather than caching the value to save gas                                    | 1         | 0.0       |
| [Gas-67]    | Write direct outcome instead of performing mathematical operations for constant state variables      | 2         | 0.0       |
| [Gas-68]    | Consider pre-calculating the address of `address(this)` to save gas                                  | 9         | 0.0       |
| [Gas-69]    | Use `storage` instead of `memory` for struct/array state variables                                   | 15        | 472500    |
| [Gas-70]    | Use constants instead of `type(uint).max`                                                            | 6         | 0.0       |
| [Gas-71]    | Setter emit which emits new and old value can be more efficient                                      | 4         | 12800     |
| [Gas-72]    | Using named returns for `pure` and `view` functions is cheaper than using regular returns            | 148       | 569504    |

---



## Conclusion

This report captures every finding explicitly mentioned in the provided reports and statements from the cantina, ensuring no omission. **Known Issues** include Pectra-related limitations and documented exclusions. **Previous Audit Report Findings** encompass all prior audits with detailed severity, status, and recommendations, totaling dozens of issues across multiple reviews. **Lightchaser Report Findings** findings from lightchaser are also outta scope. This compilation serves as a complete reference for out-of-scope items in the EigenLayer audit competition as of March 12, 2025.
