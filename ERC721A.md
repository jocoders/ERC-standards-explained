# ERC721A Gas Optimization and Cost Analysis

## How does ERC721A save gas?

ERC721A is designed to optimize gas usage, especially when minting multiple NFTs in a single transaction. The standard achieves this through three primary improvements:

### Optimization 1: Elimination of Redundant Storage

**Problem**: In the traditional ERC721 implementation (like OpenZeppelin's `ERC721Enumerable`), each token's ownership metadata is stored separately, leading to redundant storage and high gas costs.

**Solution**: ERC721A removes redundant storage by using serial token numbers starting from `0`.

```solidity
// In OpenZeppelin ERC721Enumerable
mapping(uint256 => address) private _owners;

// In ERC721A
function _removeRedundantStorage() internal {
    // Eliminate redundant storage using token serial numbers
    delete _owners; // Example code showing removal of redundant storage
}
```

### Optimization 2: Single Balance Update for Batch Minting

**Problem**: In traditional ERC721 implementations, every minted NFT updates the owner's balance, causing additional gas costs for each update.

**Solution**: ERC721A updates the balance once for the entire batch minting process, instead of updating it for each token individually.

```solidity
function _batchMint(address to, uint256 quantity) internal {
    // Update owner's balance once for all tokens
    _balances[to] += quantity; // Add the number of tokens to the owner's balance

    for (uint256 i = 0; i < quantity; i++) {
        // Mint tokens without additional balance updates
        uint256 tokenId = _nextTokenId();
        _safeMint(to, tokenId);
    }
}
```

### Optimization 3: Single Owner Data Update for Batch Minting

**Problem**: Repeatedly saving ownership data for every NFT in a batch leads to unnecessary gas usage.

**Solution**: ERC721A saves the owner data once for the first token in the batch, while subsequent tokens rely on this initial record. This approach reduces the number of storage operations required.

```solidity
function _batchMint(address to, uint256 startTokenId, uint256 quantity) internal {
    _owners[startTokenId] = to; // Set owner for the first token in the batch

    for (uint256 i = startTokenId + 1; i < startTokenId + quantity; i++) {
        // Leave the owner data empty for subsequent tokens
        _owners[i] = address(0);
    }
}

function ownerOf(uint256 tokenId) public view returns (address) {
    address owner = _owners[tokenId];
    while (owner == address(0) && tokenId > 0) {
        tokenId--;
        owner = _owners[tokenId]; // Traverse back to find the owner of the previous token
    }
    return owner;
}

```

These optimizations allow ERC721A to significantly reduce gas costs for batch minting NFTs, making the process more efficient for both developers and users.

### Data Packing and Additional Calculations

In ERC721A, ownership data and address data are packed into a single 256-bit word to save gas, but this can introduce complexity and potentially add gas costs during certain operations (like decoding and updating packed data). Let's look at how this is done:

### Data Packing for tokenId in \_packedOwnerships

ERC721A packs ownership data into a single 256-bit word, storing several fields together for each token. The fields include:

- [0..159]: Owner's address (addr), using 160 bits.
- [160..223]: Start timestamp of ownership (startTimestamp), using 64 bits.
- [224]: Burn flag (burned), using 1 bit.
- [225]: Next-initialized flag (nextInitialized), using 1 bit.
- [232..255]: Extra data (extraData), using 24 bits.

This packing reduces gas costs by minimizing the number of storage slots needed but may add some computational overhead during access and updates.

```solidity
// Example of packed ownership data
mapping(uint256 => uint256) private _packedOwnerships;

// Packing ownership details into a single 256-bit word
function _packOwnershipData(address owner, uint256 flags) internal view returns (uint256 packed) {
    packed = (uint256(uint160(owner))) // Owner's address (160 bits)
            | (block.timestamp << 160) // Start timestamp (64 bits)
            | flags;                   // Flags (like burned or nextInitialized)
}
```

### Data Packing for Address in \_packedAddressData

Similarly, owner-specific data is packed into one 256-bit word to store balances, minted tokens, and other details:

- [0..63]: Balance (balance), using 64 bits.
- [64..127]: Number of minted tokens (numberMinted), using 64 bits.
- [128..191]: Number of burned tokens (numberBurned), using 64 bits.
- [192..255]: Auxiliary data (aux), using 64 bits.

````solidity
// Example of packed address data
mapping(address => uint256) private _packedAddressData;

// Packing balance and other details into a single 256-bit word
function _packAddressData(uint64 balance, uint64 numberMinted, uint64 numberBurned, uint64 aux) internal pure returns (uint256) {
    return uint256(balance)
        | (uint256(numberMinted) << 64)
        | (uint256(numberBurned) << 128)
        | (uint256(aux) << 192);
}

Benefits of Data Packing

- Gas savings: Fewer storage reads/writes due to packing multiple fields into a single storage slot.
- Efficient storage: Maximizes use of the 256-bit storage word by packing multiple related fields.

However, this data packing introduces slight additional computational complexity during minting and retrieval processes.

## Where does ERC721A add cost?

Here’s the content formatted in Markdown with a code block, as requested:

```markdown
### Where ERC721A Increases Gas Costs

Although ERC721A is known for its efficiency in minting tokens in bulk, there are scenarios where it can lead to increased gas costs. Let's look at two main cases where this can occur, along with explanations and code examples.

### 1. Operations with a Single Token (Individual Transactions)

ERC721A is most efficient when minting tokens in bulk. However, when dealing with a single token, as in the standard ERC721, using ERC721A may not yield significant gas savings. Moreover, due to the specifics of bulk minting logic, additional checks may arise, leading to increased gas consumption.

#### Example: Minting a Single Token

In standard ERC721 when minting a single token:

```solidity
// Standard ERC721 minting
function mint(address to, uint256 tokenId) public {
    _mint(to, tokenId); // Minting a single token
}
````

Here, a basic minting operation is performed, and the contract simply records the data for one token.

In ERC721A, minting a single token may use bulk logic that is not as efficient when dealing with a small number of tokens:

```solidity
// ERC721A minting a single token
function mint(address to) public {
    _mint(to, 1); // Minting a single token using bulk minting logic
}
```

Even when minting just one token, ERC721A still performs checks intended for bulk token issuance (e.g., range checks, starting indices), which may increase gas costs.

#### Gas Comparison:

- In standard ERC721, minting a single token consumes less gas since the contract performs a simple record for one token.
- In ERC721A, despite minting only one token, the contract may perform additional operations that add slight but notable gas costs.

### 2. Complexity of Bulk Minting Logic

ERC721A introduces complex logic for optimizing bulk token minting, which can become inefficient if not utilized to its full potential. For example, if bulk minting is not frequently practiced or if minting occurs one token at a time, additional checks and records may lead to increased costs.

#### Example: Checking Starting Indices

ERC721A employs a strategy for bulk issuance that requires tracking starting indices of tokens to minimize data recording for each token. This adds a check and record for the starting index even when minting just one token:

```solidity
function _mint(address to, uint256 quantity) internal {
    uint256 startTokenId = _currentIndex; // Starting index for bulk minting

    _beforeTokenTransfers(address(0), to, startTokenId, quantity);

    for (uint256 i = 0; i < quantity; i++) {
        emit Transfer(address(0), to, startTokenId + i); // Executed even for a single token
    }

    _currentIndex += quantity; // Incrementing index even when minting one token
}
```

In the case of a bulk release (e.g., 100 tokens), this logic is highly efficient. However, when minting a single token, the contract performs the same additional steps as it would for bulk minting, potentially adding gas costs.

#### Comparison:

- **ERC721**: Minting a single token utilizes minimal checks and simple recording operations.
- **ERC721A**: The bulk minting logic performs the same steps even for a single token, adding starting index checks, tracking starting points, and possibly increasing gas expenses.

### Conclusion:

ERC721A demonstrates its efficiency in minting tokens in bulk, significantly reducing gas costs. However, when interacting with individual tokens or when a small number of tokens are minted, the ERC721A standard may add gas costs due to the bulk minting logic and additional checks that are not required in standard ERC721 contracts.

```


```
