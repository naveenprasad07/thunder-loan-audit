### [H-#] Erroneous `ThunderLoan::updateExchangeRate` in the `deposit` function causes protocol to think it has more fees than it really does, which blocks redemption and incorrectly sets the exchange rate

**Description:** In the ThunderLoan system, the `exchangeRate` is responsible for calculating the exchange rate between assetTokens and underlying tokens. In a way, it's responsible for keeping track of how many fees to give to liquidity providers.

However, the `deposit` function, updates this rate, without collecting any fees!

```solidity
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
 @>       uint256 calculatedFee = getCalculatedFee(token, amount);
 @>       assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

- Same Root Cause -> 1 finding

**Impact:** There are several impacts to this bug.

1. The `redeem` function is blocked, because the protocol thinks the owed tokens is more than it was
2. Rewards are incorrectly calculated, leading to liquidity providers potentially getting way more or less than deserved

**Proof of Concept:**

1. LP deposits
2. User takes out a flash loan
3. It is now impossible for LP to redeem

<details>
<summary>Proof of Code </summary>

Place the following into `ThunderLoanTest.t.sol`

    ```solidity
    function testRedeemAfterLoan() public setAllowedToken hasDeposits{
        uint256 amountToBorrow = AMOUNT * 10;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA,amountToBorrow);
        vm.startPrank(user);
        tokenA.mint(address(mockFlashLoanReceiver),AMOUNT);
        thunderLoan.flashloan(address(mockFlashLoanReceiver),tokenA,amountToBorrow,"");
        vm.stopPrank();

        // 1000e18
        // 3e17 fee
        // 1000e18 + 3e17j = 10003e17
        // 1003.300900000000000
        uint256 amountToRedeem = type(uint256).max;
        vm.startPrank(liquidityProvider);
        thunderLoan.redeem(tokenA,amountToRedeem);

    }

````

</details>


**Recommended Mitigation:**: Removed the incorrectly update exchange rate lines from `deposit`

```diff

  function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
-       uint256 calculatedFee = getCalculatedFee(token, amount);
-       assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
````
