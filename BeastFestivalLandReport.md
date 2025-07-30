# Beast Festival Audit Finding @CodeHawks First Flight.

### [H-1] Supply counter reset in `configurePass` allows unlimited minting beyond maximum supply limits

**Description:** 

The `FestivalPass::configurePass` function unconditionally resets the `passSupply[passId]` counter to 0, regardless of how many passes have already been minted and are currently in circulation. This creates a critical accounting mismatch between the actual number of NFTs in circulation and the contract's internal supply tracking.

```javascript
function configurePass(uint256 passId, uint256 price, uint256 maxSupply) external onlyOrganizer {
    require(passId == GENERAL_PASS || passId == VIP_PASS || passId == BACKSTAGE_PASS, "Invalid pass ID");
    require(price > 0, "Price must be greater than 0");
    require(maxSupply > 0, "Max supply must be greater than 0");

    passPrice[passId] = price;
    passMaxSupply[passId] = maxSupply;
@>  passSupply[passId] = 0; // Reset current supply - VULNERABILITY
}
```

The `buyPass` function relies on `passSupply[collectionId] < passMaxSupply[collectionId]` to enforce supply limits. When the supply counter is reset to 0, users can mint up to the full `maxSupply` amount again, completely bypassing the intended scarcity mechanism.

**Impact:** 

- **Supply Limit Bypass**: Users can mint significantly more passes than the intended maximum supply, potentially doubling or tripling the total circulation
- **Economic Fraud**: Holders who paid for "limited edition" passes are defrauded as their passes lose scarcity value
- **Revenue Loss**: The protocol may lose revenue from artificially deflated pass prices due to oversupply
- **Trust Violation**: Breaks fundamental promises about NFT scarcity and maximum supply limits
- **Market Manipulation**: Malicious organizers can intentionally flood the market by repeatedly calling `configurePass`

**Proof of Concept:**

```javascript
function testMaxSupplyBypassThroughSupplyReset() public {
    // Initial state: Max supply of 100 backstage passes
    console.log("Total max supply of Backstage pass is ", festivalPass.passMaxSupply(3));
    console.log("Total current supply of Backstage pass is ", festivalPass.passSupply(3));

    // User1 buys 50 backstage passes (50% of max supply)
    vm.startPrank(user1);
    for (uint256 i = 0; i < 50; i++) {
        festivalPass.buyPass{value: BACKSTAGE_PRICE}(3);
    }
    vm.stopPrank();

    assertEq(festivalPass.balanceOf(user1, 3), 50);
    assertEq(festivalPass.passSupply(3), 50);
    console.log("Total current supply after user purchased: ", festivalPass.passSupply(3));

    // Organizer reconfigures the pass (e.g., to update price)
    vm.prank(organizer);
    uint256 passId = 3;
    uint256 newPrice = 0.5 ether;
    uint256 newMaxSupply = 200;
    festivalPass.configurePass(passId, newPrice, newMaxSupply);

    console.log("New Total max supply of Backstage pass is ", festivalPass.passMaxSupply(3));
    console.log("New Total current supply of Backstage pass is ", festivalPass.passSupply(3));
    console.log("But User has 50 Backstage passes even though current circulation shows 0");

    // VULNERABILITY: Supply counter reset to 0 despite 50 passes in circulation
    assertTrue(festivalPass.passSupply(3) == 0); // Reset to 0
    assertEq(festivalPass.balanceOf(user1, 3), 50); // User still owns 50 passes

    // Now users can buy up to the full maxSupply again (200 more passes)
    // Total possible circulation: 50 (existing) + 200 (new) = 250 passes
    // This breaks the intended maximum supply mechanism
}
```

**Logs:**
```
Total max supply of Backstage pass is 100
Total current supply of Backstage pass is 0
Total current supply after user purchased: 50
New Total max supply of Backstage pass is 200
New Total current supply of Backstage pass is 0
But User has 50 Backstage passes even though current circulation shows 0
```

**Recommended Mitigation:** 

Remove the supply counter reset and add validation to prevent reducing max supply below current circulation:

```diff
function configurePass(uint256 passId, uint256 price, uint256 maxSupply) external onlyOrganizer {
    require(passId == GENERAL_PASS || passId == VIP_PASS || passId == BACKSTAGE_PASS, "Invalid pass ID");
    require(price > 0, "Price must be greater than 0");
    require(maxSupply > 0, "Max supply must be greater than 0");
+   require(maxSupply >= passSupply[passId], "Cannot set max supply below current supply");

    passPrice[passId] = price;
    passMaxSupply[passId] = maxSupply;
-   passSupply[passId] = 0; // Reset current supply
}
```

This ensures that:
1. The supply counter accurately reflects the actual number of passes in circulation
2. Maximum supply cannot be reduced below the current circulating supply
3. The scarcity mechanism works as intended
4. Existing pass holders' investments are protected


### \[H-2] Cannot Redeem Maximum Supply of Memorabilia NFTs

**Description:**

The `FestivalPass::redeemMemorabilia` function is intended to allow users to redeem a fixed-supply NFT from a memorabilia collection. However, the current logic prevents users from redeeming the final (i.e., `maxSupply`-th) NFT due to an off-by-one error in this condition:

```javascript
    // Redeem a memorabilia NFT from a collection
    function redeemMemorabilia(uint256 collectionId) external {
        MemorabiliaCollection storage collection = collections[collectionId];
@>      require(collection.currentItemId < collection.maxSupply, "Collection sold out"); // VULNERABILITY
```

This check disallows redemption when `currentItemId == maxSupply - 1` (just before the last mint) and then **prevents** minting when `currentItemId == maxSupply`, thereby allowing only `maxSupply - 1` NFTs to be minted.

**Impact:**

* The collection's `maxSupply` is never fully reachable.
* It results in an under-allocation of NFTs, which could affect the scarcity and value perception of the collection.
* Users expecting to be able to redeem up to the defined `maxSupply` will face unexpected reverts.

**Proof of Concept:**

```javascript
function testCannotRedeemMaxMemoriableNft() public {
    // Setup: User buys VIP pass and gets bonus BEAT
    vm.prank(user1);
    festivalPass.buyPass{value: VIP_PRICE}(2); // Gets 5e18 BEAT bonus

    // Create performance and let user attend to earn BEAT
    vm.prank(organizer);
    uint256 perfId = festivalPass.createPerformance(block.timestamp + 1 hours, 2 hours, 250e18);

    vm.warp(block.timestamp + 90 minutes);
    vm.prank(user1);
    festivalPass.attendPerformance(perfId); // Earns 500e18 BEAT

    // Organizer creates a memorabilia collection with maxSupply = 5
    vm.prank(organizer);
    uint256 collectionId = festivalPass.createMemorabiliaCollection(
        "Festival Poster",
        "ipfs://QmPosters",
        100e18,
        5,
        true
    );

    // User redeems 4 out of 5 allowed NFTs successfully
    vm.startPrank(user1);
    festivalPass.redeemMemorabilia(collectionId); // #1
    festivalPass.redeemMemorabilia(collectionId); // #2
    festivalPass.redeemMemorabilia(collectionId); // #3
    festivalPass.redeemMemorabilia(collectionId); // #4

    // Attempt to redeem the 5th (final) NFT reverts
    vm.expectRevert("Collection sold out");
    festivalPass.redeemMemorabilia(collectionId); // #5 (should succeed, but fails)
}
```

**Root Cause:**

The condition `collection.currentItemId < collection.maxSupply` prevents the minting of the final item. When `currentItemId == maxSupply - 1`, the redemption succeeds and increments the ID to `maxSupply`, at which point further redemptions are blocked—even though the total supply is now exactly equal to the allowed maximum.


**Recommended Mitigation:**

Update the condition to be less than and equal to the `collection.maxSupply`

```diff
- require(collection.currentItemId < collection.maxSupply, "Collection sold out");
+ require(collection.currentItemId <= collection.maxSupply, "Collection sold out");
```

Or Alternatively we can start the `currentItemId` index at 0 in the `FestivalPass::createMemorabiliaCollection` function

```diff
        collections[collectionId] = MemorabiliaCollection({
            name: name,
            baseUri: baseUri,
            priceInBeat: priceInBeat,
            maxSupply: maxSupply,
-           currentItemId: 1, // Start item IDs at 1
+           currentItemId: 1, // Start item IDs at 0
            isActive: activateNow
        });
```
that way we can always mint up to the maxSupply of the NFT collection.