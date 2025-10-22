  
        This project demonstrates a PoC for the Cooldown reset issue which leads for Temporary freezing of funds.


Here's Poc to run 

1)Setup foundry and set path use the full path: `~/.foundry/bin/forge`.
2}From ..contracts/test/foundry/staking , run

'''
~/./.foundry/bin/forge build
~/.foundry/bin/forge test --match-contract CooldownResetVulnerabilityPOC -vvv

'''

The test file is located at `bbp-public-assets/contracts/test/foundry/staking/PROOF_OF_CONCEPT_COOLDOWN_RESET.sol` and contains the complete, replicable PoC.


### Vulnerable Code

```solidity
// StakedUSDeV2.sol:96-105
function cooldownAssets(uint256 assets) external ensureCooldownOn returns (uint256 shares) {
    if (assets > maxWithdraw(msg.sender)) revert ExcessiveWithdrawAmount();
    shares = previewWithdraw(assets);
    
    // ❌ VULNERABILITY: Resets cooldown for ALL assets
    cooldowns[msg.sender].cooldownEnd = uint104(block.timestamp) + cooldownDuration;
    cooldowns[msg.sender].underlyingAmount += uint152(assets); // Accumulates amounts
    
    _withdraw(msg.sender, address(silo), msg.sender, assets, shares);
}

// StakedUSDeV2.sol:109-118
function cooldownShares(uint256 shares) external ensureCooldownOn returns (uint256 assets) {
    if (shares > maxRedeem(msg.sender)) revert ExcessiveRedeemAmount();
    assets = previewRedeem(shares);
    
    // ❌ VULNERABILITY: Same issue - resets cooldown for ALL assets
    cooldowns[msg.sender].cooldownEnd = uint104(block.timestamp) + cooldownDuration;
    cooldowns[msg.sender].underlyingAmount += uint152(assets);
    
    _withdraw(msg.sender, address(silo), msg.sender, assets, shares);
}



##Poc

image.png


### Real-World Attack Scenario

```
Timeline:
────────────────────────────────────────────────────
Day 0:   User cooldowns 1,000 USDe 
         → Expected unlock: Day 90
         
Day 85:  User cooldowns 500 USDe
         → Cooldown RESET for ALL 1,500 USDe
         → New unlock: Day 175 from day 0
         
Day 90:  User attempts withdrawal of 1,000 USDe
         → ❌ REVERTS (cooldown not complete)
         → Expected: Should work ✅
         
Day 175: User finally able to withdraw
         → ✅ SUCCESS (all 1,500 USDe)

         and so on.........
────────────────────────────────────────────────────


Impact:
• Original funds locked for 85 extra mpre days,
• During market volatility = real economic loss
• Users cannot stage withdrawals
