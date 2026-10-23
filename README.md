# On-chain Verifiable Randomness (VRF): Integration Guide

**English** | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md)

A verifiable randomness service on BNB Smart Chain mainnet. Your contract sends a request; an oracle generates the random number with a BLS12-381 unique signature, the chain verifies it, and your contract receives the result in a callback. The seed includes the hash of the request block, so nobody can predict the result when the request is sent, and anyone can re-verify it afterwards.

- Network: **BNB Smart Chain mainnet (chainId 56)**
- Payment: native **BNB**
- Two ways to integrate: **Subscription** (prepaid, billed at actual cost) or **Pay-per-request** (attach BNB to each request)

## Table of contents

1. [Contract addresses](#addresses)
2. [Parameters and limits](#params)
3. [Quick start](#quickstart)
   - [3.1 Define the interfaces](#interfaces)
   - [3.2 Option 1: Subscription](#subscription)
   - [3.3 Option 2: Pay-per-request](#payperrequest)
4. [Fees](#fees)
5. [Callback rules and safe usage](#rules)
6. [Verifying afterwards](#verify)
7. [Common errors](#errors)
8. [Operator guide (run an oracle node)](#operators)
   - [8.1 Overview and admission](#op-overview)
   - [8.2 Checklist](#op-prepare)
   - [8.3 Download and verify the node software](#op-download)
   - [8.4 Generate keys and apply](#op-register)
   - [8.5 Configure and run the node](#op-run)
   - [8.6 Monitoring and alerts](#op-monitor)
   - [8.7 Day-to-day operations](#op-ops)
   - [8.8 Earnings and withdrawal](#op-earn)
   - [8.9 Obligations and slashing](#op-slash)
   - [8.10 Exiting](#op-exit)
9. [Limitations](#limits)
10. [License](#license)

<a id="addresses"></a>
## 1. Contract addresses (BNB Smart Chain mainnet, chainId 56)

| Contract | Address | Purpose |
|---|---|---|
| Coordinator | `0x481044B907C40A8eF406691168064BCe8B3fb836` | Subscriptions, requests, fulfilment, verification |
| PayPerRequest | `0x72AA07A2c1852F6EdB1Ee068ca1Af74444A8A5Ac` | Pay-per-request entry point |
| Reader | `0x2F7d603EA5Fac589701177f2D570F0417F37bf30` | Balances, quotes, re-verifying random numbers |
| HeaderArchive | `0x3366b0cBBf31DBCf4dC1EC4dc87F844a4490A4d6` | Stores request block hashes (for later re-verification) |
| OperatorRegistry | `0x01DA9bfe6b614f9497c97efE3C90Dfb172fB45Ca` | Operator registration |
| Verifier | `0x46F8c7579B6A299750341237bAaeA6fFbf7A2334` | BLS signature verification |

**Official lane:** `0xfe6f6c68fa6c50e1569c6fe61d9637af93d04632c000c835d41464b41efef2d0`

A lane identifies one oracle key (the `keccak256` of its public key). You must specify a lane when you send a request. The Coordinator and PayPerRequest are upgradeable proxies, so **always use the proxy addresses above**.

<a id="params"></a>
## 2. Parameters and limits

| Item | Value |
|---|---|
| Minimum confirmations | 3 (you may use more, up to 200) |
| Random words per request | up to 500 |
| Consumer contracts per subscription | up to 100 |
| Callback gas limit | 2,500,000 |
| Request timeout (cancelled if unfulfilled) | 100,000 blocks (about 12.5 hours at the current block time) |
| Fulfilment batch | up to 25 requests per transaction |

See [Fees](#fees) for the rates.

<a id="quickstart"></a>
## 3. Quick start

<a id="interfaces"></a>
### 3.1 Define the interfaces (no extra dependencies)

```solidity
interface IRandomnessCoordinator {
    function createSubscription() external returns (uint256 subId);
    function fundSubscription(uint256 subId) external payable;
    function addConsumer(uint256 subId, address consumer) external;
    function cancelSubscription(uint256 subId, address payable to) external;
    function request(bytes32 lane, uint256 subId, uint16 confirmations, uint32 callbackGas, uint32 numWords)
        external returns (uint256 requestId);
    function quote(bytes32 lane, uint32 callbackGas, uint32 numWords) external view returns (uint256);
}

interface IPayPerRequest {
    function quote(uint32 callbackGas, uint32 numWords) external view returns (uint256);
    function request(uint32 callbackGas, uint32 numWords) external payable returns (uint256 requestId);
}
```

Your contract must implement two callbacks (callable only by the Coordinator / PayPerRequest):

```solidity
function onRandomness(uint256 requestId, uint256[] calldata words) external;
function onRequestCancelled(uint256 requestId) external; // optional; called when a timed-out request is cancelled
```

<a id="subscription"></a>
### 3.2 Option 1: Subscription (many requests, frequent use)

```solidity
contract MyGame {
    address constant COORDINATOR = 0x481044B907C40A8eF406691168064BCe8B3fb836;
    bytes32 constant LANE = 0xfe6f6c68fa6c50e1569c6fe61d9637af93d04632c000c835d41464b41efef2d0;
    uint256 public subId;
    mapping(uint256 => address) public playerOf;

    // 1. One-time setup: create a subscription, fund it, register this contract as a consumer
    //    (you can also do this from a wallet)
    function setup() external payable {
        IRandomnessCoordinator c = IRandomnessCoordinator(COORDINATOR);
        subId = c.createSubscription();
        c.fundSubscription{value: msg.value}(subId);
        c.addConsumer(subId, address(this));
    }

    // 2. Send a request: 12 confirmations, 200,000 callback gas, 1 random word
    function play() external {
        uint256 id = IRandomnessCoordinator(COORDINATOR).request(LANE, subId, 12, 200_000, 1);
        playerOf[id] = msg.sender;
    }

    // 3. Callback: only the Coordinator may call it
    function onRandomness(uint256 requestId, uint256[] calldata words) external {
        require(msg.sender == COORDINATOR, "only coordinator");
        // use words[0]; do not revert; do not write a lot of storage here
    }

    function onRequestCancelled(uint256 requestId) external {
        require(msg.sender == COORDINATOR, "only coordinator");
        // The request timed out and was cancelled: void this round / refund. Do NOT request again here.
    }
}
```

The example is simplified to show the flow and omits access control; add your own access control to `setup`, withdrawals and similar functions.

Any address can create and fund a subscription; the creator is the subscription owner. When you no longer need it, the owner calls `cancelSubscription` to withdraw the balance (not possible while requests are pending).

<a id="payperrequest"></a>
### 3.3 Option 2: Pay-per-request (occasional use, no subscription to manage)

```solidity
address constant PPR = 0x72AA07A2c1852F6EdB1Ee068ca1Af74444A8A5Ac;

function play() external payable {
    uint256 price = IPayPerRequest(PPR).quote(200_000, 1);   // callback gas, number of random words
    uint256 id = IPayPerRequest(PPR).request{value: price}(200_000, 1); // any overpayment is refunded immediately
}
// The random numbers are forwarded to the calling contract's onRandomness(requestId, words)
```

If the callback fails, the random numbers are stored and can be delivered again with PayPerRequest's `retry(requestId)`. They are not lost.

<a id="fees"></a>
## 4. Fees

- **Subscription:** when you send a request, the worst-case cost at the lane's maximum gas price is **reserved** (not charged). On fulfilment you are charged the **actual cost**, never more than the reservation; the rest stays in the subscription.
  Charge = actual gas cost × (1 + premium) + flat fee
- **Pay-per-request:** price = Coordinator quote × (1 + markup); any overpayment is refunded immediately.
- **Rates**

| Item | Value | Notes |
|---|---|---|
| Premium | **10%** | Added on top of the actual gas cost |
| Flat fee | **0.001 BNB** | Charged once per fulfilment |
| Pay-per-request markup | **10%** | Pay-per-request only, added on top of the Coordinator quote |

- **Read on chain (the chain is authoritative):**
  - Premium `premiumBps` and flat fee `flatFee`: `Coordinator.settings()`
  - Markup `markupBps`: `PayPerRequest.markupBps()`
- **Quotes and balance checks:**
  - `Coordinator.quote(lane, callbackGas, numWords)`: how much one request reserves
  - `PayPerRequest.quote(callbackGas, numWords)`: the pay-per-request price at the current gas price
  - `Reader.affordable(subId, lane, callbackGas, numWords)`: how many more requests the subscription can afford
  - `Reader.shortfall(subId, lane, callbackGas, numWords, n)`: how many wei short you are to send n requests

A larger callback gas limit, more random words and a higher gas price all increase the cost.

<a id="rules"></a>
## 5. Callback rules and safe usage

1. **Do not revert** in the callback and do not exceed the `callbackGas` you requested. Otherwise the delivery is recorded as failed and must be delivered again.
2. **Do not write a lot of storage** in the callback. Store a seed or a result digest and recompute the rest in view functions.
3. **Lock the relevant state** between sending the request and receiving the callback. Do not let users decide whether to take part after they have seen the result.
4. For high-value draws, use **12 or more confirmations** (protection against chain reorganisations).
5. A timed-out request is cancelled and `onRequestCancelled` is called. Void the round or refund there. **Do not request again**; that would re-roll with a new seed.
6. You may send new requests from inside the callback (consecutive draws).

<a id="verify"></a>
## 6. Verifying afterwards

Anyone can call `Reader.verifyFulfilment(lane, requestId, blockNumber, numWords, sig)` to recompute and check a random number. After each fulfilment the node stores the request block hash in HeaderArchive, so requests older than 256 blocks can still be verified.

<a id="errors"></a>
## 7. Common errors

| Symptom | Cause and fix |
|---|---|
| `Underfunded` | The subscription's available balance is below the reservation. Top up, or use `Reader.shortfall` to compute the gap |
| `Underpaid` | Pay-per-request: `msg.value` is below the quote. Call `quote` first |
| Callback not triggered | Check the `DeliveryFailed` event. Subscription mode: anyone can call `redeliver`. Pay-per-request: call `retry` |
| Request not fulfilled for a long time | After the timeout (100,000 blocks) it is cancelled and your `onRequestCancelled` is called |
| Callback rejected | The callback only accepts calls from the Coordinator / PayPerRequest |

<a id="operators"></a>
## 8. Operator guide (run an oracle node)

<a id="op-overview"></a>
### 8.1 Overview and admission

Any team can apply to become an operator, run its own oracle node and earn a share of the service fees. Your node holds one BLS key and produces a unique signature for each request; the Coordinator verifies the signature on chain and then calls the consumer contract. **The private key never goes on chain and is never shared.**

- Admission: submit your public key and proof of possession, post the bond, and **wait for approval**.
- Bond: **minimum 2 BNB**, returned when you exit (minus any slashing).
- Earnings: **70%** of the service fee (default, up to 90%) plus the gas compensation for fulfilment.

<a id="op-prepare"></a>
### 8.2 Checklist

| Item | Recommendation |
|---|---|
| Server | Linux, **1 CPU core / 1 GB RAM / 10 GB disk** is enough; Docker installed; always on |
| RPC | **At least 2 different providers**, both must support `getLogs` (public nodes usually do not); keep `GETLOGS_RANGE` within the provider's per-call limit (commonly 5000) |
| BLS key | Generated locally, stored encrypted, **never put on chain, never shared** |
| Hot wallet | A separate, **independent** key used only to send fulfilment transactions; **start with 0.1 BNB**, top up when it falls below 0.01 BNB (gas is compensated by the protocol, but you front it) |
| Admin address | The address used to apply and to withdraw the bond (a hardware wallet or multisig works); different from the hot wallet and the BLS key |
| Bond | 2 BNB |

The two keys (BLS key and hot wallet) **must be different**; the node checks this at startup.

<a id="op-download"></a>
### 8.3 Download and verify the node software

1. Download `vrf-oracle-<version>.tar.gz` and `SHA256SUMS` from this repository's [Releases](../../releases).
2. Verify integrity (in the directory containing both files):

```bash
tar xzf vrf-oracle-1.1.0.tar.gz
sha256sum -c SHA256SUMS      # macOS: shasum -a 256 -c SHA256SUMS
```

3. Build the image:

```bash
cd vrf-oracle-1.1.0
docker build -t vrf-oracle:1.1.0 .
```

The image contains no keys; keys, passwords and RPC URLs are provided at run time. Run the following commands in your working directory (for example `~/vrf`).

<a id="op-register"></a>
### 8.4 Generate keys and apply

**Step 1: generate the BLS key** (the password is typed in the terminal, at least 12 characters, not echoed; the private key exists only in the encrypted file)

```bash
mkdir -p keys && sudo chown 1000:1000 keys
docker run --rm -it -v "$PWD/keys:/keys" --entrypoint node vrf-oracle:1.1.0 \
  opcli.mjs keygen --keystore /keys/oracle.keystore
```

**Step 2: save the password file and the hot-wallet key file** (typed with `read -s`, so nothing enters your shell history)

```bash
read -rs -p "keystore password: " P; printf '%s' "$P" > ks_password; unset P; echo
read -rs -p "hot wallet private key (0x...): " K; printf '%s' "$K" > tx_key; unset K; echo
sudo chown 1000:1000 ks_password tx_key && sudo chmod 0400 ks_password tx_key
```

**Step 3: show the public key and lane**

```bash
docker run --rm -it -v "$PWD/keys:/keys:ro" --entrypoint node vrf-oracle:1.1.0 \
  opcli.mjs pubkey --keystore /keys/oracle.keystore
```

**Step 4: generate the application.** First write a minimal `reg.env`:

```
RPC_URLS=https://your-rpc-1,https://your-rpc-2
COORDINATOR=0x481044B907C40A8eF406691168064BCe8B3fb836
EXPECT_CHAIN_ID=56
```

```bash
docker run --rm -it --env-file reg.env -v "$PWD/keys:/keys:ro" --entrypoint node vrf-oracle:1.1.0 \
  opcli.mjs register --keystore /keys/oracle.keystore \
  --registry 0x01DA9bfe6b614f9497c97efE3C90Dfb172fB45Ca \
  --admin <your-admin-address> --payee <payout-address> --value 2
```

By default it **only prints** the call data (`applyOperator(pk, pop)` with 2 BNB attached) and sends nothing. We recommend sending the transaction from your admin address's wallet (hardware wallet or multisig): to = the registry address, value = 2 BNB, data = the printed call data. Alternatively, set `ADMIN_KEY_FILE` to a file holding the admin private key and add `--send` to send from the command line (`--admin` must match the address of that key).

**Step 5: wait for approval.** After you apply, the status is Pending. Once approved, the registry registers your lane on the Coordinator. Use the health check (see 8.5) to see the status.

<a id="op-run"></a>
### 8.5 Configure and run the node

Create `node.env`:

```
RPC_URLS=https://your-rpc-1,https://your-rpc-2
COORDINATOR=0x481044B907C40A8eF406691168064BCe8B3fb836
EXPECT_CHAIN_ID=56
EXPECTED_LANE=<the lane from step 3>
GETLOGS_RANGE=5000
POLL_MS=5000
TX_KEY_FILE=/run/secrets/tx_key
ORACLE_KEYSTORE=/keys/oracle.keystore
ORACLE_KEYSTORE_PASSWORD_FILE=/run/secrets/ks_password
```

**Run the health check before starting** (do not start if anything FAILs):

```bash
docker run --rm -it --env-file node.env \
  -v "$PWD/keys:/keys:ro" -v "$PWD/ks_password:/run/secrets/ks_password:ro" -v "$PWD/tx_key:/run/secrets/tx_key:ro" \
  --entrypoint node vrf-oracle:1.1.0 opcli.mjs doctor
```

The health check covers: RPC connectivity, `getLogs` availability, the Coordinator, hot-wallet balance, lane and public key consistency, the archive contract, and the registry status (Pending / Active). **Start only when the registry status is Active and all checks pass.**

**Start the node:**

```bash
docker run -d --name vrf-oracle --restart unless-stopped --env-file node.env \
  -v "$PWD/keys:/keys:ro" \
  -v "$PWD/ks_password:/run/secrets/ks_password:ro" \
  -v "$PWD/tx_key:/run/secrets/tx_key:ro" \
  -v vrf-state:/data \
  --read-only --cap-drop ALL --security-opt no-new-privileges \
  vrf-oracle:1.1.0
```

Check that it is running:

```bash
docker ps --filter name=vrf-oracle        # STATUS should be healthy
docker logs --tail 20 vrf-oracle          # you should see: node up, lane ...
```

The node refreshes its cursor file (in the `vrf-state` volume) for every new block and resumes from the last position after a restart, so no requests are missed. After each fulfilment it stores the request block hash in HeaderArchive for later verification.

<a id="op-monitor"></a>
### 8.6 Monitoring and alerts

The same image includes a read-only monitor (no keys). Create `monitor.env`:

```
RPC_URLS=https://your-rpc-1,https://your-rpc-2
COORDINATOR=0x481044B907C40A8eF406691168064BCe8B3fb836
REGISTRY=0x01DA9bfe6b614f9497c97efE3C90Dfb172fB45Ca
LANES=<your lane>
NODE_WALLET=<hot wallet address>
NODE_MIN_BALANCE_WEI=10000000000000000
PENDING_ALERT_BLOCKS=400
GETLOGS_RANGE=5000
POLL_MS=60000
MONITOR_NAME=my-oracle
ALERT_WEBHOOK=
```

```bash
docker run -d --name vrf-monitor --restart unless-stopped --env-file monitor.env \
  -v vrf-monitor-state:/data -e STATE_FILE=/data/vrf-monitor-state.json \
  --read-only --cap-drop ALL --entrypoint node vrf-oracle:1.1.0 monitor.mjs
```

It raises an alert when: the hot-wallet balance drops below `NODE_MIN_BALANCE_WEI` (0.01 BNB), a request stays unfulfilled for more than `PENDING_ALERT_BLOCKS` blocks, or the lane is disabled or contract state looks abnormal. Alerts are written to the log (`docker logs vrf-monitor`; lines with `level=alert` need action). If you set `ALERT_WEBHOOK`, alerts are also pushed there as a JSON POST (fields: `source, name, level, ev, ts, text, detail`; `text` is a one-line summary, so you can forward it to Telegram, Slack, WeCom and similar).

<a id="op-ops"></a>
### 8.7 Day-to-day operations

- **Stay online.** A request that is still unfulfilled more than 8000 blocks (about 1 hour at the current block time) after it becomes ready can be reported and slashed (see [8.9](#op-slash)). Keep the node running with `--restart unless-stopped`, and keep any upgrade or migration downtime under 10 minutes.
- **Upgrade the node:**
  1. Download the new release, verify `SHA256SUMS`, and run `docker build -t vrf-oracle:<new-version> .`
  2. `docker stop vrf-oracle && docker rm vrf-oracle`
  3. Start the new image with the same options and the **same `vrf-state` volume**; the cursor carries over.
- **Hot-wallet balance.** Top up when it falls below 0.01 BNB; do not wait until it cannot send transactions.
- **RPC.** Use at least two providers and watch free-tier quotas (a 3000–5000 ms poll interval is enough; a shorter interval burns quota quickly).
- **Backups.** Keep offline backups of `keys/oracle.keystore` and the password file. **Losing the BLS key means that lane can no longer serve**; you would need to generate a new key and apply again.
- **Replacing the hot wallet.** Update the `tx_key` file and recreate the container. The lane is unaffected.

<a id="op-earn"></a>
### 8.8 Earnings and withdrawal

- On each fulfilment: `service fee = amount charged − gas compensation`, and your share = service fee × share ratio (default 70%). The share is credited to your **payout address** as `credits`; the gas compensation is credited to the **address that submitted the fulfilment transaction** (the hot wallet).
- Withdraw by calling `Coordinator.claimCredits(to)` from the corresponding address, for example from that wallet on BscScan's Write Contract page.
- The share ratio is configured per lane; changes are not retroactive.

<a id="op-slash"></a>
### 8.9 Obligations and slashing

| Item | Value |
|---|---|
| Minimum bond | 2 BNB |
| Counted as missed | Not fulfilled more than 8000 blocks (about 1 hour at the current block time) after the request is ready; anyone may report |
| Penalty per incident | 0.1 BNB |
| Minimum gap between two penalties for one operator | 1 hour (one incident is penalised once) |
| Slashing cap | Within a rolling 7 days, at most 20% of the bond |
| Dispute period | 2 days (the arbiter may veto; after it ends anyone can execute the slash) |
| Automatic suspension | 3 penalties within 7 days |
| Unbonding period | 14 days (starts when you request exit; then you can `withdraw`) |
| Lane maximum gas price | Range 0.1 to 2 gwei; a new lane defaults to 1 gwei |

- Slashed funds do not go to the protocol; they are sent to the burn address.
- Only requests whose block hash is available are penalised (last 256 blocks, or archived). A request that could not be fulfilled because a node was offline too long is not penalised.
- After a suspension, top up the bond (`topUp(id)`) and the approver reinstates you.
- The values above follow `OperatorRegistry.params()` on chain.

<a id="op-exit"></a>
### 8.10 Exiting

1. Call `OperatorRegistry.requestExit(id)` from the admin address (look up `id` with `opOfLane(lane)`). From then on no new requests are accepted.
2. Wait for the unbonding period (14 days). The bond can still be reduced by pending slashes.
3. When the period ends, call `OperatorRegistry.withdraw(id, to)` to recover the bond.
4. Before exiting, let the node finish its in-flight requests, then stop it.

<a id="limits"></a>
## 9. Limitations

- BNB Smart Chain mainnet only; payment in BNB only.
- The interface is not compatible with Chainlink; existing Chainlink contracts must be adapted to the interface above.
- A single operator can refuse to fulfil. A request that times out is cancelled and your contract is notified, so it never fails silently. If your application cannot tolerate delayed randomness, do not depend on a single lane.

<a id="license"></a>
## 10. License

The documentation and the node software in this repository (`vrf-oracle-<version>.tar.gz` in Releases) are released under the **MIT License**, copyright © 2026 clkhoo5211. The full text is in [LICENSE](LICENSE).

- You may use, modify, sell and redistribute them freely, as long as you keep the copyright and license notices.
- The software and documentation are provided "as is", without warranty of any kind, express or implied. You use the service and the software at your own risk.
- The node's third-party dependencies (`ethers`, `@noble/curves`) are each under their own MIT licenses.
- The contract addresses listed here are public information on BNB Smart Chain mainnet.
