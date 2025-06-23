### [H-1] Global `s_earnTimer` in `Snow::earnSnow` restricts earning to a single user per week

**Description:**  
The `Snow::earnSnow` function uses a single global timestamp (`s_earnTimer`) to enforce a one-week cooldown for minting new tokens. This means that once one user calls `Snow::earnSnow` function, all other users are prevented from earning Snow tokens until the cooldown expires. This central design flaw severely restricts usability and fairness across the protocol.

```javascript 
function earnSnow() external canFarmSnow {
    if (s_earnTimer != 0 && block.timestamp < (s_earnTimer + 1 weeks)) {
        revert S__Timer();
    }
    _mint(msg.sender, 1);

    // Uses a global timer — once any user calls earnSnow, no one else can earn for a week
@>   s_earnTimer = block.timestamp;
}
```

**Impact:** This limits the earning of snow tokens to only the first user who calls the `Snow::earnSnow` during a given week will be able to earn tokens. All other users will be blocked until the global timer resets, leading to unfair token distribution 

**Proof of Concept:**
1. Ashley call the `Snow::earnSnow` function
2. The `s_earnTimer` is updated to the current timestamp.
3. Jerry tries to call the `Snow::earnSnow` function but it fails because current `s_earnTimer` is less than the 1 week duration

**Proof of Code**
<details>
<summary>Code</summary>

```javascript
    function testOnlySingleUserCanEarnSnowWeekly() public {
        vm.prank(ashley);
        snow.earnSnow();
        vm.stopPrank();

        vm.prank(jerry);
        vm.expectRevert();
        snow.earnSnow(); // This would revert because s_earnTimer is set to the current timestamp after ashley minted
        vm.stopPrank();
    }

```
</details>


**Recommended Mitigation:** To fix this, we should use a mapping instead to keep track of timestamp for each individual users that calls the `Snow::earnSnow` instead;
```diff
- uint256 private s_earnTimer;
+ mapping(address => uint256) private s_earnTimer;

    function earnSnow() external canFarmSnow {
-       if (s_earnTimer != 0 && block.timestamp < (s_earnTimer + 1 weeks)) {
+       if (s_earnTimer[msg.sender] != 0 && block.timestamp < (s_earnTimer[msg.sender] + 1 weeks)) {
            revert S__Timer();
        }
        _mint(msg.sender, 1);
-       s_earnTimer = block.timestamp;
+ s_earnTimer[msg.sender] = block.timestamp:
    }

```

### [H-2] Missing Staking Verification Allows Unrestricted Minting of Snowman NFTs

**Description:**
The `Snowman::mintSnowman` function allows unrestricted minting of Snowman NFTs without verifying if the receiver is a Snow token staker. This directly contradicts the intended design where only Snow token stakers should receive these NFTs, potentially allowing unlimited unauthorized NFT creation.

**Impact:** This allows any user to mint an unlimited number of Snowman NFTs without staking Snow tokens, directly violating the tokenomics and fairness of the system.

**Proof of Concept:**
1. Any user can call Snowman::mintSnowman directly.
2. The caller can specify any recipient address and any amount to be minted.
3. NFTs are minted without any staking verification.

**Recommended Mitigation:** Introduce staking verification and minting limitations to enforce proper eligibility checks:
1. Add a cap on the maximum number of NFTs a user can mint in a single call.
2. Import the Snow token contract into Snowman to verify staking balances.
3. Ensure that the receiver holds the minimum required Snow tokens before minting.

```diff
+   uint256 private constant MAX_MINT_AMOUNT;
+   uint256 private constant MIN_STAKING_AMOUNT;

+    Snow public immutable snow; // Snow token contract
+   constructor(address snowAddress) {
+       snow = Snow(snowAddress);
+    }

    function mintSnowman(address receiver, uint256 amount) external {
        for (uint256 i = 0; i < amount; i++) {
+     // Check if receiver has minimum required Snow tokens
+     require(snow.balanceOf(receiver) >= MIN_STAKING_AMOUNT, "S__InsufficientSnow");
+     require(amount <= MAX_MINT_AMOUNT, "S__ExceedsMaxMint");
+           s_TokenCounter++; // follow CEI to prevent reentrancy attacks
            _safeMint(receiver, s_TokenCounter);

            emit SnowmanMinted(receiver, s_TokenCounter);

-            s_TokenCounter++;
        }
    }
```

### [H-3] Nft token drain in `SnowmanAirdrop::claimSnowman()` due to missing claim check.

**Description:**
The `SnowmanAirdrop::claimSnowman()` function includes a mapping `s_hasClaimedSnowman[receiver] = true` to track which addresses have already claimed the airdrop. However, the function fails to check this mapping is `true/false` before allowing a claim. As a result, eligible users can repeatedly call the function and mint multiple NFTs.

```solidity
function claimSnowman(address user, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s) external {
    ...
@>     s_hasClaimedSnowman[receiver] = true;
}
```

**Impact:**
Any qualified user can call the `SnowmanAirdrop::claimSnowman()` function multiple times minting an unlimited amount of nfts till the contract is drained or deflates the price of the Nft.


**Proof of Concept:**

1. Alice calls the `SnowmanAirdrop::claimSnowman()` function.
2. Alice calls the `SnowmanAirdrop::claimSnowman()` function again and succed
3. Alice now has **multiple NFTs claimed** from a single intent.

**Proof of Code:**

<details>
<summary>Test Code</summary>

```javascript
    function testClaimMultipleTimes() public {
        assert(nft.balanceOf(alice) == 0);
        vm.startPrank(alice);
        snow.approve(address(airdrop), type(uint256).max);

        bytes32 alDigest = airdrop.getMessageHash(alice);
        (uint8 alV, bytes32 alR, bytes32 alS) = vm.sign(alKey, alDigest);

        airdrop.claimSnowman(alice, AL_PROOF, alV, alR, alS);
        assert(nft.balanceOf(alice) == 1);
        assert(nft.ownerOf(0) == alice);

        uint256 FEE = snow.s_buyFee();
        snow.buySnow{value: FEE}(1); // Alice then buy 1 more snow token to match the signature proof

        airdrop.claimSnowman(alice, AL_PROOF, alV, alR, alS);
        assert(nft.balanceOf(alice) == 2);
    }
```
```javascript
Ran 1 test for test/TestSnowmanAirdrop.t.sol:TestSnowmanAirdrop
[PASS] testClaimMultipleTimes() (gas: 265735)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.35ms (1.22ms CPU time)
```

</details>

**Recommended Mitigation:**
To fix this add a check at the top of `SnowmanAirdrop::claimSnowman()` function to check if a user has claimed or not.

```diff
+   error SA__UserAlreadyClaimed();

    function claimSnowman(address receiver, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s)
        external
        nonReentrant
    {
+       if (s_hasClaimedSnowman[receiver]) {
            revert SA__UserAlreadyClaimed();
        } else {
            s_hasClaimedSnowman[receiver] = true;
        }

        if (receiver == address(0)) {
            revert SA__ZeroAddress();
        }
        if (i_snow.balanceOf(receiver) == 0) {
            revert SA__ZeroAmount();
        }

        if (!_isValidSignature(receiver, getMessageHash(receiver), v, r, s)) {
            revert SA__InvalidSignature();
        }

        uint256 amount = i_snow.balanceOf(receiver);

        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(receiver, amount))));

        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert SA__InvalidProof();
        }

        i_snow.safeTransferFrom(receiver, address(this), amount); // send tokens to contract... akin to burning

-        s_hasClaimedSnowman[receiver] = true;

        emit SnowmanClaimedSuccessfully(receiver, amount);

        i_snowman.mintSnowman(receiver, amount);
    }
```

### [H-4] Merkle Proof Verification Uses Current Token Balance Instead of Snapshot Amount, Breaking Airdrop Eligibility

**Description:**
The `SnowmanAirdrop::claimSnowman()` function contains a critical flaw in how it reconstructs the Merkle tree leaf during verification. It uses the **user’s live Snow token balance** at the time of claiming instead of a **fixed amount** that was originally committed in the Merkle tree:

```javascript
@>  uint256 amount = i_snow.balanceOf(receiver);
    bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(receiver, amount))));
```

This introduces a major inconsistency. The Merkle tree was generated off-chain using a **static snapshot of eligible addresses and their corresponding amounts**. However, if a user’s balance has changed between that snapshot and the actual claim — even by 1 token — the computed leaf will not match the original Merkle proof, and the claim will fail.

**Impact:**

* Eligible users can be **unfairly denied** their airdrop claim due to **minor balance changes** (e.g. additional purchases, transfers, or earnings).
* Delegated claim systems become unreliable, as users cannot be guaranteed to maintain the exact balance until a relayer submits the claim.
* This breaks the determinism of Merkle-based verification and can severely degrade user experience and trust.

**Proof of Concept:**

1. Merkle tree was generated assigning each eligible user with amount = 1.
2. Alice has 5 snow token during claiming period 
3. Alice calls the `claimSnowman()` function and it fails due to `SA__InvalidProof`.

**Proof of Code**
<details>
<summary>Test Code</summary>

```javascript
    function testClaimingOfAirdropDeniedDueToSnowBalanceGreaterThanOne() public {
        assert(nft.balanceOf(alice) == 0);
        vm.startPrank(alice);
        uint256 FEE = snow.s_buyFee();
        snow.buySnow{value: FEE * 10}(10); // alice buys 10 snow tokens
        snow.approve(address(airdrop), type(uint256).max);
        bytes32 alDigest = airdrop.getMessageHash(alice);
        (uint8 alV, bytes32 alR, bytes32 alS) = vm.sign(alKey, alDigest);

        // Reverts because alice can only claim one snow NFt but the proof use dynamic token balance instead.
        vm.expectRevert(SnowmanAirdrop.SA__InvalidProof.selector);
        airdrop.claimSnowman(alice, AL_PROOF, alV, alR, alS);
    }
```
```solidity
    Ran 1 test for test/TestSnowmanAirdrop.t.sol:TestSnowmanAirdrop
    [PASS] testClaimingOfAirdropDeniedDueToSnowBalanceGreaterThanOne() (gas: 104546)
    Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.49ms (714.92µs CPU time)

    │   ├─ [573] Snow::balanceOf(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6]) [staticcall]
    │   │   └─ ← [Return] 11
    │   └─ ← [Revert] SA__InvalidProof()
    └─ ← [Return] 
```
</details>

**Recommended Mitigation:**

1. Make the Amount Explicit in the Signature

Instead of dynamically computing the amount, we **explicitly pass the `amount`** to the `getMessageHash` function. This ensures that the signed message reflects the correct, fixed claim amount.

```diff
- function getMessageHash(address receiver) public view returns (bytes32) {
+ function getMessageHash(address receiver, uint256 amount) public view returns (bytes32) {
    if (i_snow.balanceOf(receiver) == 0) {
        revert SA__ZeroAmount();
    }

-    uint256 amount = i_snow.balanceOf(receiver);

    return _hashTypedDataV4(
        keccak256(abi.encode(MESSAGE_TYPEHASH, SnowmanClaim({receiver: receiver, amount: amount})))
    );
}
```
2. Update the Claim Function to Accept Amount

We also need to update the `claimSnowman` function to accept the `amount` parameter and pass it through to `getMessageHash`. This removes reliance on dynamic state within the verification logic.

```diff
- function claimSnowman(address receiver, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s)
+ function claimSnowman(address receiver, uint256 amount, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s) {
    
    // rest of the logic...

- if (!_isValidSignature(receiver, getMessageHash(receiver), v, r, s)) {
+ if (!_isValidSignature(receiver, getMessageHash(receiver, amount), v, r, s)) {
    revert SA__InvalidSignature();
}

bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(receiver, amount))));

if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
    revert SA__InvalidProof();
}
```

**Alternative Solution: Enable Partial Claims**

For a more flexible and user-friendly design, we can support **partial claims** by introducing a mapping that tracks how much each user has already claimed. This prevents users from claiming more than they’re entitled to while allowing them to claim in multiple transactions.

#### 🔁 Add Tracking for Claimed Amounts

```solidity
mapping(address => uint256) public s_claimedAmount;
```

* Users can claim multiple times up to their **maximum entitled amount**.
* The system validates that `claimed + new amount ≤ entitlement`.

### [M-1] Replay Attack Vulnerability in `SnowmanAirdrop::claimSnowman()`

**Description:**
The `SnowmanAirdrop::claimSnowman()` function is vulnerable to replay attacks due to the insecure construction of the signed message hash in `getMessageHash()`. The hash includes a dynamic `amount` value based on the user's current snow token balance which can be manipulated after the signature is created. This allows an attacker to manipulate the balance and reuse the same valid signature multiple times, enabling unauthorized or repeated airdrop claims. 

```javascript
    function getMessageHash(address receiver) public view returns (bytes32) {
        if (i_snow.balanceOf(receiver) == 0) {
            revert SA__ZeroAmount();
        }

@>       uint256 amount = i_snow.balanceOf(receiver);

        return _hashTypedDataV4(
            keccak256(abi.encode(MESSAGE_TYPEHASH, SnowmanClaim({receiver: receiver, amount: amount})))
        );
    }
```

**Impact:**
An attacker can reuse the same valid signed message from a user multiple times, allowing **unauthorized control**. This can:

* Drain user's Snow token balances repeatedly.
* Inflate NFT supply.
* Break intended single-claim logic.

**Proof of Concept:**

1. Alice signs a message to allow an airdrop claim.
2. Satoshi calls `SnowmanAirdrop::claimSnowman` with Alice's signature — succeeds.
3. An attacker **calls the same function again** with **same signature** — succeeds again.
4. Alice now has **multiple NFTs claimed** from a single intent.

This is possible because the message hash includes `amount`, which is dynamic and changes depending on current token balance which can be manipulate.

**Proof of Code:**

<details>
<summary>Test Code</summary>

```javascript
    function testReplayAttackByRelayer() public {
        console2.log("Testing Replay Attack on claimSnowman");
        assert(nft.balanceOf(alice) == 0);
        vm.prank(alice);
        snow.approve(address(airdrop), type(uint256).max);

        // Get alice's digest and signed message
        bytes32 alDigest = airdrop.getMessageHash(alice);
        (uint8 alV, bytes32 alR, bytes32 alS) = vm.sign(alKey, alDigest);

        console2.log("Alice snow token balance before satoshi claim for her", snow.balanceOf(alice));
        console2.log("Alice nftToken balance before satoshi claim for her", nft.balanceOf(alice));

        // satoshi calls claims on behalf of alice using her signed message to claim her airdrop for her
        vm.prank(satoshi);
        airdrop.claimSnowman(alice, AL_PROOF, alV, alR, alS);
        
        console2.log("Alice snow token balance after satoshi claim for her", snow.balanceOf(alice));
        console2.log("Alice nftToken balance after satoshi claim for her", nft.balanceOf(alice));
        assert(nft.balanceOf(alice) == 1);
        assert(nft.ownerOf(0) == alice);

        // assuming alice now buy/earn snow token after sastoshi have claimed the airdrop
        // or Attacker transfer snow token to alice address
        vm.startPrank(alice);
        uint256 FEE = snow.s_buyFee();
        snow.buySnow{value: FEE}(1);
        assertEq(snow.balanceOf(alice), 1); // since amount is dynamic it can be manupulated
        vm.stopPrank();

        console2.log("ReplayAttak: Alice snow token balance before satoshi claim for her", snow.balanceOf(alice));
        console2.log("ReplayAttack: Alice nftToken balance before satoshi claim for her", nft.balanceOf(alice));

        // Attacker calls claims on behalf of alice using her signed message
        vm.prank(attacker);
        airdrop.claimSnowman(alice, AL_PROOF, alV, alR, alS);
        assert(nft.balanceOf(alice) == 2);

        console2.log("ReplayAttak: Alice snow token balance AFTER satoshi claim for her", snow.balanceOf(alice));
        console2.log("ReplayAttack: Alice nftToken balance AFTER satoshi claim for her", nft.balanceOf(alice));
    }
```
```javascript
Ran 1 test for test/TestSnowmanAirdrop.t.sol:TestSnowmanAirdrop
    [PASS] testReplayAttackByRelayer() (gas: 292482)
Logs:
  Testing Replay Attack on claimSnowman
  Alice snow token balance before satoshi claim for her 1
  Alice nftToken balance before satoshi claim for her 0

  Alice snow token balance after satoshi claim for her 0
  Alice nftToken balance after satoshi claim for her 1

  ReplayAttak: Alice snow token balance before satoshi claim for her 1
  ReplayAttack: Alice nftToken balance before satoshi claim for her 1

  ReplayAttak: Alice snow token balance AFTER satoshi claim for her 0
  ReplayAttack: Alice nftToken balance AFTER satoshi claim for her 2

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.14ms (1.35ms CPU time)
```

</details>

**Recommended Mitigation:**
To prevent signature replay we need to make these changes.

1. Add a `nonce` and `deadline` in the `SnowmanAirdrop::SnowmanClaim` struct to make it more replay resistant.

```diff
    struct SnowmanClaim {
        address receiver;
        uint256 amount;
+       uint256 deadline;
+       uint256 nonce
    }
```
2. Update the `MESSAGE_TYPEHASH` to the newly update struct.

```diff
- bytes32 private constant MESSAGE_TYPEHASH = keccak256("SnowmanClaim(addres receiver, uint256 amount)");
+ bytes32 public constant MESSAGE_TYPEHASH = keccak256("SnowmanClaim(address receiver,uint256 amount,uint256 deadline,uint256 nonce)");
```
2. Add a mapping to keep track of the nonce if it has been used or not.

```diff
// Now, we also need to keep track of nonces!
// Track used nonces to prevent replay
+ mapping(address => mapping(uint256 => bool)) public noncesUsed;

```
3. Update the `getMessageHash` , `_isValidSignature` and `claimSnowman` function.

```diff
-    function getMessageHash(address receiver) public view returns (bytes32) {
+     function getMessageHash(address receiver, uint256 deadline, uint256 nonce) public view returns (bytes32) {
        if (i_snow.balanceOf(receiver) == 0) {
            revert SA__ZeroAmount();
        }

        uint256 amount = i_snow.balanceOf(receiver);

        return _hashTypedDataV4(
            keccak256(
                abi.encode(
                    MESSAGE_TYPEHASH,
                    receiver,
                    amount,
+                   deadline,
+                   nonce
                )
            )
        );
    }
```

```diff
-    function _isValidSignature(address receiver, bytes32 digest, uint8 v, bytes32 r, bytes32 s)
+ -    function _isValidSignature(address receiver, uint256 deadline, uint256 nonce bytes32 digest, uint8 v, bytes32 r, bytes32 s)
        internal
        pure
        returns (bool)
    {
+       require(noncesUsed[receiver][nonce], "Nonced used already!");
+       noncesUsed[receiver][nonce] = true; 
+       require(block.timestamp < deadline, "Expired");
        (address actualSigner,,) = ECDSA.tryRecover(digest, v, r, s);
        return actualSigner == receiver;
    }
```
```diff
-    function claimSnowman(address receiver, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s)
        external
        nonReentrant
    {
+    function claimSnowman(address receiver, uint256 deadline, uint256 nonce, bytes32[] calldata merkleProof, uint8 v, bytes32 r, bytes32 s)
        external
        nonReentrant
    {
-       if (!_isValidSignature(receiver, getMessageHash(receiver), v, r, s)) {
           revert SA__InvalidSignature();
        }
+        if (!_isValidSignature(receiver, getMessageHash(receiver, deadline, nonce), v, r, s)) {
            revert SA__InvalidSignature();
        }
```
This updated implementation mitigates replay attacks by validating that the `nonce` is unused and the `deadline` has not passed before accepting any claim.

