# low-risk-reports

# Low Risk

## 1. Don’t use `payable.transfer()`/`payable.send()`

_The use of `payable.transfer()` is [heavily frowned upon](https://consensys.net/diligence/blog/2019/09/stop-using-soliditys-transfer-now/) because it can lead to the locking of funds. The `transfer()` call requires that the recipient is either an EOA account, or is a contract that has a `payable` callback. For the contract case, the `transfer()` call only provides 2300 gas for the contract to complete its operations. This means the following cases can cause the transfer to fail:_

-   The contract does not have a `payable` callback.
-   The contract’s `payable` callback spends more than 2300 gas (which is only enough to emit something).
-   The contract is called through a proxy which itself uses up the 2300 gas Use OpenZeppelin’s `Address.sendValue()` instead.
-   Future changes or forks in Ethereum result in higher gas fees than transfer provides. The .transfer() creates a hard dependency on 2300 gas units being appropriate now and into the future.

Example wrong:

```java
payable(payAddress).transfer(payAmt);
```

Instead use the `.call()` function to transfer ether and avoid some of the limitations of .`transfer()`;

Example:

```java
(bool success, ) = payable(payAddress).call{value: payAmt}(""); // royalty transfer to royaltyaddress
require(success, "Transfer failed.");
```

Note: Gas units can also be passed to the `.call()` function as a variable to accomodate any uses edge cases. Gas could be a mutable state variable that can be set by the contract owner.

<br>
<hr>

## 2. Unused/empty `receive()`/`fallback()` function

_If the intention is for the Ether to be used, the function should call another function, otherwise it should revert (e.g. `require(msg.sender == address(weth))`)_

Example:

```java
fallback() external payable {}
receive() external payable {}
```

<br>
<hr>

## 3. Use Custom Errors instead of String

_Use custom errors to save deployment and runtime costs in case of revert._

_Instead of using strings for error messages (e.g., `require(msg.sender == owner, “unauthorized”)`), you can use custom errors to reduce both deployment and runtime gas costs. In addition, they are very convenient as you can easily pass dynamic information to them._

_Starting from `Solidity v0.8.4`, there is a convenient and gas-efficient way to explain to users why an operation failed through the use of custom errors. Until now, you could already use strings to give more information about failures like this:_

Before:

```java
function add(uint256 _amount) public {
    require(msg.sender == owner, "unauthorized");

    total += _amount;
}
```

```java
 require(shares + ONE_DEC18 < totalShares, "too many shares");
```

After:

```java
error Unauthorized(address caller);

function add(uint256 _amount) public {
    if (msg.sender != owner)
        revert Unauthorized(msg.sender);

    total += _amount;
}
```

<br>
<hr>

## 4. `require()` should be used instead of `assert()`

_Prior to solidity version 0.8.0, hitting an assert consumes the remainder of the transaction’s available gas rather than returning it, as `require()`/`revert()`._

_`assert()` should be avoided even past solidity version 0.8.0 as its [documentation](https://docs.soliditylang.org/en/v0.8.14/control-structures.html#panic-via-assert-and-error-via-require) states that “The assert function creates an error of type Panic(uint256). … Properly functioning code should never create a Panic, not even on invalid external input. If this happens, then there is a bug in your contract which you should fix”._

Example:

```java
493:          assert(idToOwner[_tokenId] == address(0));
506:          assert(idToOwner[_tokenId] == _from);
```

<br>
<hr>

## 5. Unsafe use of `transfer()`/`transferFrom()` with `IERC20`

_Some tokens do not implement the ERC20 standard properly but are still accepted by most code that accepts ERC20 tokens. For example Tether (USDT)'s `transfer()` and `transferFrom()` functions do not return booleans as the specification requires, and instead have no return value. When these sorts of tokens are cast to `IERC20`, their function signatures do not match and therefore the calls made, revert. Use OpenZeppelin’s SafeERC20's `safeTransfer()`/`safeTransferFrom()` instead_

[Example](https://github.com/code-423n4/2022-05-rubicon/blob/8c312a63a91193c6a192a9aab44ff980fbfd7741/contracts/rubiconPools/BathPair.sol#L601):

```java
File: contracts/rubiconPools/BathPair.sol

601:              IERC20(asset).transfer(msg.sender, booty);

615:              IERC20(quote).transfer(msg.sender, booty);
```

<br>
<hr>

## 6. Return values of `transfer()`/`transferFrom()` not checked

_Not all `IERC20` implementations `revert()` when there's a failure in `transfer()`/`transferFrom()`. The function signature has a `boolean` return value and they indicate errors that way instead. By not checking the return value, operations that should have marked as failed, may potentially go through without actually making a payment_

[Example](https://github.com/code-423n4/2022-05-rubicon/blob/8c312a63a91193c6a192a9aab44ff980fbfd7741/contracts/rubiconPools/BathPair.sol#L601): (same code as before)

```java
File: contracts/rubiconPools/BathPair.sol

601:              IERC20(asset).transfer(msg.sender, booty);

615:              IERC20(quote).transfer(msg.sender, booty);

```

<br>
<hr>

## 7. Return values of `approve()` not checked

_Not all `IERC20` implementations `revert()` when there's a failure in `approve()`. The function signature has a `boolean` return value and they indicate errors that way instead. By not checking the return value, operations that should have marked as failed, may potentially go through without actually approving anything_

[Example:](https://github.com/code-423n4/2022-05-rubicon/blob/8c312a63a91193c6a192a9aab44ff980fbfd7741/contracts/rubiconPools/BathToken.sol#L214)

```java
File: contracts/rubiconPools/BathToken.sol   #1

214:   IERC20(address(token)).approve(RubiconMarketAddress, 2**256 - 1);
```

Recommended Mitigation Steps:

-   It is recommend to use `safeApprove()`.
-   Consider using `safeIncreaseAllowance` instead of approve function. (Approve race condition)

<br>
<hr>

## 8. Typos

Example:

```java
/// @audit usefull
60:           uint256 nonce; // nonce of order usefull for cancelling in bulk
```

<br>
<hr>

## 9. NatSpec is incomplete

```java
File: contracts/core/GolomTrader.sol

/// @audit Missing: '@return'
162       ///      OrderStatus = 3 , valid order
163       /// @param o the Order struct to be validated
164       function validateOrder(Order calldata o)
165           public
166           view
167           returns (
168               uint256,
169               bytes32,
170:              uint256

/// @audit Missing: '@param tokenId'
/// @audit Missing: '@param proof'
328       /// @dev function to fill a signed order of ordertype 2 also has a payment param in case the taker wants
329       ///      to send ether to that address on filling the order, Match an criteria order, ensuring that the supplied proof demonstrates inclusion of the tokenId in the associated merkle root, if root is 0 then any token can be used to fill the order
330       /// @param o the Order struct to be filled must be orderType 2
331       /// @param amount the amount of times the order is to be filled(useful for ERC1155)
332       /// @param referrer referrer of the order
333       /// @param p any extra payment that the taker of this order wanna send on succesful execution of order
334       function fillCriteriaBid(
335           Order calldata o,
336           uint256 amount,
337           uint256 tokenId,
338           bytes32[] calldata proof,
339           address referrer,
340           Payment calldata p
341:      ) public nonReentrant {
```

## 9.1 Missing NatSpec in file.

<br>
<hr>

## 10. Inconsistent spacing in comments

_Some lines use `// x` and some use `//x`. The instances below point out the usages that don’t follow the majority, within each file_

Example:

```java
File: contracts/core/GolomTrader.sol
181:          //deadline

File: contracts/rewards/RewardDistributor.sol
99:           //console.log(block.timestamp,epoch,fee);
```

## 11. Missing checks for `address(0x0)` (zero-address) when assigning values to `address` state variables

_Missing checks for zero-addresses may lead to infunctional protocol, if the variable addresses are updated incorrectly._
Example:

`factory = _factory;`

```java
constructor(
        address _factory,
        INonfungiblePositionManager _nonfungiblePositionManager,
        ISwapRouter _swapRouter,
        ComptrollerInterface _comptroller
    ) {
        factory = _factory;
        nonfungiblePositionManager = _nonfungiblePositionManager;
        swapRouter = _swapRouter;
        comptroller = _comptroller;
        _notEntered = true;
    }
```

<br>

`pendingDistributor = _distributor;`

```java
/// @notice Sets the distributor contract
    /// @param _distributor Address of the distributor
    function setDistributor(address _distributor) external onlyOwner {
        if (address(distributor) == address(0)) {
            distributor = Distributor(_distributor);
        } else {
            pendingDistributor = _distributor;
            distributorEnableDate = block.timestamp + 1 days;
        }
    }
```

Recommended Mitigation Steps:

-   Consider adding zero-address checks:
    `require(newAddr != address(0));`

<br>
<hr>

## 12. The Contract Should `approve(0)`/`safeApprove()` first - A second call to the function may revert

_Some ERC20 tokens, such as Tether, `revert()` if `approve()` is called a second time without having called `approve(0)` first_

_Some tokens (like USDT L199) do not work when changing the allowance from an existing non-zero allowance value.
They must first be approved by zero and then the actual allowance must be approved._

_When trying to re-approve an already approved token, all transactions revert and the protocol cannot be used._

Recommended Mitigation Steps:

-   Approve with a zero amount first before setting the actual amount. Consider use `safeIncreaseAllowance` and `safeDecreaseAllowance`.

Example:

```java
IERC20(token).safeApprove(address(operator), 0);
IERC20(token).safeApprove(address(operator), amount);
```

<br>
<hr>

## 13. Chainlink's VRF V1 is deprecated

_VRF V1 is [deprecated](https://docs.chain.link/vrf/v1/introduction), so new projects should not use it._

```java
13   /// @notice RandProvider wrapper around Chainlink VRF v1.
14:  contract ChainlinkV1RandProvider is RandProvider, VRFConsumerBase {
```

<br>
<hr>
<br>

based on real reports [Code4arena](https://code4rena.com/reports)
