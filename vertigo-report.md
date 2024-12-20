### Mutation Testing Report Summary

- **Number of mutations:** 102
- **Mutations killed:** 78/102
- **Mutations survived (Lived):** 24/102

### Key Mutations Discovered and Results

#### 1. **Condition Mutation**
   - **File:** `StakingManager.sol`
   - **Line:** 66
   - **Original Code:** 
     ```solidity
     if (stakings[tokenId] != 0) revert TokenAlreadyStaked(tokenId);
     ```
   - **Mutated Code:**
     ```solidity
     if (stakings[tokenId] == 0) revert TokenAlreadyStaked(tokenId);
     ```
   - **Result:** Killed
   - **Insight:** Condition mutation changed `!=` to `==`, resulting in invalid staking logic, which was caught in testing.

#### 2. **Loop Condition Mutation**
   - **File:** `Merkle.sol`
   - **Line:** 55
   - **Original Code:** 
     ```solidity
     while (data.length > 1) {
     ```
   - **Mutated Code:**
     ```solidity
     while (data.length <= 1) {
     ```
   - **Result:** Killed
   - **Insight:** Altering the loop condition from `>` to `<=` would prevent the loop from running as expected, which was identified in tests.

#### 3. **Bitwise Operation Mutation**
   - **File:** `Merkle.sol`
   - **Line:** 95
   - **Original Code:** 
     ```solidity
     if (length & 0x1 == 1) {
     ```
   - **Mutated Code:**
     ```solidity
     if (length | 0x1 == 1) {
     ```
   - **Result:** Killed
   - **Insight:** Changing the bitwise `&` to `|` modifies the logical conditions for length checking, which caused issues detected by tests.

#### 4. **Access Control Mutation**
   - **File:** `StakingManager.sol`
   - **Line:** 38
   - **Original Code:** 
     ```solidity
     function acceptRewardTokenOwnership() external onlyOwner {
     ```
   - **Mutated Code:**
     ```solidity
     function acceptRewardTokenOwnership() external {
     ```
   - **Result:** Killed
   - **Insight:** Removing `onlyOwner` modifies access control, allowing unauthorized access, which was identified in testing.

#### 5. **Arithmetic Operation Mutation**
   - **File:** `StakingManager.sol`
   - **Line:** 107
   - **Original Code:** 
     ```solidity
     reward = (block.timestamp - timestamp) * REWARD_PER_SECOND;
     ```
   - **Mutated Code:**
     ```solidity
     reward = (block.timestamp + timestamp) * REWARD_PER_SECOND;
     ```
   - **Result:** Killed
   - **Insight:** Mutation introduced incorrect reward calculation, which was caught by tests validating expected reward outputs.

#### 6. **Mutation Resulting in Errors**
   - **File:** `LimitedEditionNFT.sol`
   - **Line:** 47
   - **Original Code:** 
     ```solidity
     Ownable(msg.sender)
     ```
   - **Mutated Code:** (Removed `Ownable`)
   - **Result:** Error
   - **Insight:** Removing `Ownable` constructor call caused an error, affecting contract ownership setup.

#### 7. **Survived Mutations (Lived)**
   - **Examples:**
     - **File:** `Merkle.sol`, **Line:** 133
       - Original: `if (x <= 1) {`
       - Mutation: `if (x < 1) {`
     - **File:** `Merkle.sol`, **Line:** 171
       - Original: `if ((lsb == _x) && (msb > 0)) {`
       - Mutation: `if ((lsb == _x) && (msb <= 0)) {`
   - **Insight:** These mutations survived, indicating that either the tests didn’t cover these cases or the mutated logic didn’t affect test outcomes. Adding or improving tests may be necessary to capture such mutations.


