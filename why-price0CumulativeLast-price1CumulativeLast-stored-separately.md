### Why are `price0CumulativeLast` and `price1CumulativeLast` stored separately? Why not just calculate `price1CumulativeLast = 1 / price0CumulativeLast`?

`price0CumulativeLast` and `price1CumulativeLast` are stored separately in Uniswap V2 to reflect **independent cumulative pricing for each token pair**, and there are several reasons why calculating `price1CumulativeLast` as `1 / price0CumulativeLast` would be problematic:

1. **Lack of Reciprocal Dependence**:
   - In Uniswap V2, each token has its own cumulative price measure: `price0CumulativeLast` accumulates the price of token1 in terms of token0, while `price1CumulativeLast` accumulates the price of token0 in terms of token1.
   - These cumulative values grow independently, as they are based on the **specific reserves and time** associated with each token. Since prices can fluctuate independently for each token, calculating one as the inverse of the other would not accurately capture this independent accumulation.

2. **Precision Issues**:
   - Solidity does not support floating-point numbers, which creates challenges when dealing with reciprocal calculations. Instead, Uniswap V2 uses a fixed-point representation (`UQ112x112`) to maintain high precision.
   - If the contract tried to calculate `price1CumulativeLast` as `1 / price0CumulativeLast`, it could lead to a significant **loss of precision** and **overflow risks**, especially when the token prices differ widely.

3. **Efficiency and Simplified Calculations**:
   - With each reserve update (via the `_update` function), the contract updates both cumulative prices based on the current reserves. Directly storing both cumulative values simplifies the calculations, allowing for straightforward TWAP calculations.
   - This means that when calculating TWAP, each cumulative value can be subtracted independently without needing to calculate one through an indirect or inverse value, leading to simpler and more efficient logic.

4. **Independence for External Contracts and Integrations**:
   - Many external projects and contracts may require only `price0CumulativeLast` or only `price1CumulativeLast` depending on their specific use case. For instance, a contract might only be interested in the price of token0 in terms of token1 over a period, and thus, it would only need `price1CumulativeLast`.
   - Storing these values separately allows other contracts to access each cumulative price independently, avoiding unnecessary calculations or transformations.

### Conclusion:
Storing `price0CumulativeLast` and `price1CumulativeLast` separately enables Uniswap V2 to track price changes independently and with high precision. This approach avoids issues with inverse calculations, simplifies contract logic, and provides greater flexibility for other applications and contracts that interact with Uniswap’s pricing data.
