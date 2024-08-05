### [H-1] Erroneous `ThunderLoan::updateExchangeRate` in `deposit` to thik it has more feees than it really does, which blocks redemption and incorrectly sets the exchange rate.

**Description:** In the ThunderLoan system, the `exchangeRate` is responsible for calculating the exchange rate between assetTokens and underlying tokens. In a way, its responsible for keeping track of how many fees to give to liquidity providers.

However, the `deposit` function, updates this rate, without collecting any fees

```javascript
   function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
@>      uint256 calculatedFee = getCalculatedFee(token, amount);
@>      assetToken.updateExchangeRate(calculatedFee);
        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

**Impact:** There are several impacts to this bug

1. The `redeem` function is blocked, because the protocol thinks the owed is more than it has
2. Rewards are incorrectly calculated, leading to liquidity providers potentially getting way more or less than deserved.

**Proof of Concept:**

1. LP deposits
2. User takes out a flash loan
3. It is now impossible for LP to redeem

<details>

<summary>Proof of Code</summary>

```javascript
    function testRedeemAfterLoan() public setAllowedToken() hasDeposits() {
        uint256 amountToBorrow = AMOUNT * 10;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);
        vm.startPrank(user);
        tokenA.mint(address(mockFlashLoanReceiver), calculatedFee);
        thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, amountToBorrow, "");
        vm.stopPrank();

        uint256 amountToRedeem = type(uint256).max;
        vm.startPrank(liquidityProvider);
        thunderLoan.redeem(tokenA,amountToRedeem);
    }
```

</details>

**Recommended Mitigation:** Remove the incorrectly updated exchange rate 
```diff
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);
-      uint256 calculatedFee = getCalculatedFee(token, amount);
-      assetToken.updateExchangeRate(calculatedFee);
       token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```


### [H-2] Mixing up variabvle location causes storage collisions in `ThunderLoan::s_flashLoanFee` and `ThunderLoan::s_currentlyFlashLoaning`, freezing protocol.

**Description:** `ThunderLoan.sol` has two variable in the following other.

```javascript
    uint256 private s_feePrecision;
    uint256 private s_flashLoanFee; // 0.3% ETH fee
    mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```
However, the upgraded contract `ThunderLoanUpgraded.sol` has them in a different order.

```javascript
    uint256 private s_flashLoanFee; // 0.3% ETH fee
    uint256 public constant FEE_PRECISION = 1e18;
    mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
```
Due to how solidity works, after the upgrade the `s_flashLoanFee` will have the value of `s_precision`. You cannot adjust the position of storage variables, and removing storage variables for constant variables, breaks the stprage locations as well.

**Impact:**  After upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. This means that users who take out flash loans right after an upgrade will be charged the wrong fee. 
Additionally the `s_currentlyFlashLoaning` mapping will start on the wrong storage slot.

**Proof of Concept:**
<details>
<summary>PoC</summary>

Place the following into `ThunderLoanTest.t.sol`/

```javascript
    import { ThunderLoanUpgraded } from "../../src/upgradedProtocol/ThunderLoanUpgraded.sol";

    function testUpgradeBreaks() public {
        uint256 feeBeforeUpgrade = thunderLoan.getFee();
        vm.startPrank(thunderLoan.owner());
        ThunderLoanUpgraded upgraded = new ThunderLoanUpgraded();
        thunderLoan.upgradeToAndCall(address(upgraded), "");
        uint256 feeAfterUpgrade = thunderLoan.getFee();
        vm.stopPrank();
        console.log("feeBeforeUpgrade", feeBeforeUpgrade);
        console.log("feeAfterUpgrade", feeAfterUpgrade);
        assert(feeBeforeUpgrade != feeAfterUpgrade);
    }
```
You can also see the storage layout difference by running `forge inspect ThunderLoan storage` and `forge inspect ThunderLoanUpgraded storage`

</details>

**Recommended Mitigation:** Do not switch the positions of the storage variables on upgrade, and leave a blank if you're going to replace a storage variable with a constant. In `ThunderLoanUpgraded.sol:`
```diff
-    uint256 private s_flashLoanFee; // 0.3% ETH fee
-    uint256 public constant FEE_PRECISION = 1e18;
+    uint256 private s_blank;
+    uint256 private s_flashLoanFee; 
+    uint256 public constant FEE_PRECISION = 1e18;
```