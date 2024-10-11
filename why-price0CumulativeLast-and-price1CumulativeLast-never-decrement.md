### Why does `price0CumulativeLast` and `price1CumulativeLast` never decrement?

In Uniswap V2, `price0CumulativeLast` and `price1CumulativeLast` are part of the cumulative price mechanism used to calculate average prices over time. These variables are updated and stored as cumulative values that represent the integral of the price over time.

#### Explanation:

1. **Cumulative Price Mechanism**:
   - Uniswap V2 uses cumulative prices (`price0CumulativeLast` and `price1CumulativeLast`) to track the integral of the product of token prices and time. Every time there is a state change (like a swap or liquidity addition/removal), these values are updated based on the time elapsed since the last update.

2. **Accumulation of Price Changes**:
   - When a trade occurs, the contract calculates how much time has passed since the last state update (`blockTimestampLast`). It then updates `price0CumulativeLast` and `price1CumulativeLast` by multiplying the current reserve ratios (`UQ112x112.encode(_reserve1).uqdiv(_reserve0)` for `price0CumulativeLast` and vice versa for `price1CumulativeLast`) with the elapsed time. This operation accumulates the changes in the price over time.

3. **Why They Never Decrement**:
   - Decrementing these values would imply that the integral of price over time has decreased, which doesn't align with the intended behavior of accumulating prices. In a cumulative mechanism, the values only increase or remain unchanged over time as more transactions occur and update the reserves.

4. **Usage in TWAP Calculation**:
   - To calculate the Time Weighted Average Price (TWAP) over a specific period, Uniswap V2 subtracts the cumulative prices at two different points in time and divides by the time elapsed between these points. This provides a smoothed average price that reflects market conditions over the given period.
   
   The formula for TWAP is:
   
   **TWAP = (priceCumulativeLast - priceCumulativeLast) / timeElapsed**
   where:
   - priceCumulativeLast is the cumulative price at the end of the observation period.
   - priceCumulativeLast is the cumulative price at the beginning of the observation period.
   - timeElapsed is the time in seconds between the start and end of the observation period.

#### Code Example:

```solidity
function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
    require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');
    uint32 blockTimestamp = uint32(block.timestamp % 2**32);
    uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
    if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
        // * never overflows, and + overflow is desired
        price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
        price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
    }
    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
    blockTimestampLast = blockTimestamp;
    emit Sync(reserve0, reserve1);
}
```

In the `_update` function above:
- `price0CumulativeLast` and `price1CumulativeLast` are updated based on the time elapsed since the last update and the current reserve ratios.
- The `timeElapsed` variable ensures that the cumulative price values are adjusted continuously based on the passage of time and the changes in reserves.

### Conclusion:

The design of `price0CumulativeLast` and `price1CumulativeLast` ensures that they only increase or stay the same over time, reflecting the cumulative effect of price changes in the Uniswap V2 pair contract. This mechanism supports accurate TWAP calculations and maintains consistency in pricing data across different blocks and transactions.
