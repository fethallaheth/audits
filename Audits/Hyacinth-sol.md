
---

### H1: **Incorrect Type in `hasRole` Function Call (Bytes Expected, String Provided)**

#### Summary:
The `onlyNotBlacklisted` modifier uses the `hasRole` function, passing a string `"BLACKLISTED_ROLE"` where a bytes32 type is expected, leading to incorrect functionality when checking roles.

#### Description:
The `hasRole` function in the contract expects the first argument to be of `bytes32` type, but the `onlyNotBlacklisted` modifier passes `"BLACKLISTED_ROLE"` as a string. This mismatch could cause the function to revert unexpectedly or fail to correctly identify blacklisted addresses. Using incorrect data types may break the role-based access control system, leading to unexpected permission handling.

#### Impact:
This issue prevents the contract from properly identifying blacklisted addresses, which may allow blacklisted entities to interact with the contract and potentially expose the system to malicious actors.

#### Mitigation:
Replace the string `"BLACKLISTED_ROLE"` with the correct bytes32 type, such as:
```solidity
bytes32 constant BLACKLISTED_ROLE = keccak256("BLACKLISTED_ROLE");
```
Then, pass `BLACKLISTED_ROLE` to the `hasRole` function in the `onlyNotBlacklisted` modifier:
```solidity
if (hasRole(BLACKLISTED_ROLE, _address))
    revert BlacklistedAddress(_address);
```

--- 
---

### H2: **No Address is Blacklisted Due to Missing `setBlacklist` Function**

#### Summary:
The contract lacks a `setBlacklist` function to assign the "BLACKLISTED_ROLE" to addresses, rendering the blacklist mechanism ineffective.

#### Description:
While the contract includes a `onlyNotBlacklisted` modifier that checks if an address holds the `BLACKLISTED_ROLE`, there is no function implemented to assign this role to blacklisted addresses. Without a mechanism to blacklist addresses, the modifier's role-based access control cannot be enforced, leaving the system vulnerable to unauthorized interactions by addresses that should be restricted.

#### Impact:
The absence of the `setBlacklist` function prevents any address from being blacklisted, allowing malicious or unwanted entities to interact with the contract without any restriction. This could lead to significant security and operational risks, depending on the role of the blacklist in the contract's functionality.

#### Mitigation:
Implement a `setBlacklist` function that allows authorized entities to assign the `BLACKLISTED_ROLE` to specific addresses. Example:
```solidity
function setBlacklist(address _address) external {
    _setupRole(BLACKLISTED_ROLE, _address);
}
```
This function should also have access control to ensure that only privileged roles (e.g., admin) can blacklist addresses.


---
---

### H3: **`buy` Function Not Marked as `payable` When Using `msg.value`**

#### Summary:
The `buy` function references `msg.value`, but it is not marked as `payable`, preventing the function from accepting ETH transfers.

#### Description:
The `buy` function is designed to allow users to purchase tokens by sending ETH, using `msg.value` to determine the amount of ETH transferred. However, the function is not marked as `payable`, which means it cannot receive ETH. Any attempt to send ETH in a transaction to this function will result in a failure, preventing users from participating in the token sale.

#### Impact:
The function will revert whenever a user tries to send ETH to purchase tokens, rendering the token sale mechanism inoperative and blocking potential sales.

#### Mitigation:
Mark the `buy` function as `payable` to allow the contract to accept ETH:
```solidity
function buy() public payable onlyNotBlacklisted(msg.sender) returns (uint saleTokenAmount) {
```
This will enable the contract to receive ETH and correctly handle token purchases based on the value of `msg.value`.



--- 
---


### H-3: **Lack of Access Control in `updateFairLaunchProperties`**

#### Summary:
The `updateFairLaunchProperties` function lacks access control, allowing any user to modify critical properties of the contract.

#### Description:
The function currently permits any external user to update the `fairLaunchProperties` variable, which may include sensitive configurations for the contract. Without a proper access control mechanism (such as a modifier to restrict access to the contract owner), any malicious actor could alter these properties, leading to potential vulnerabilities and undesired behaviors.

#### Impact:
Unauthorized users could modify crucial parameters, such as tax rates or launch status, resulting in the risk of loss of funds or exploitation of the contract.

#### Mitigation:
Implement an access control mechanism to restrict the function to the contract owner:
```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "Caller is not the owner");
    _;
}
```
Apply this modifier to the `updateFairLaunchProperties` function:
```solidity
function updateFairLaunchProperties(FairLaunchProperties memory _fairLaunchProperties) external onlyOwner returns (bool) {
    fairLaunchProperties = _fairLaunchProperties;
    return true;
}
```
This change ensures that only authorized personnel can update critical properties of the contract.

---
---


### H-4: **Incorrect Price Calculation Formula in `getPrice`**

#### Summary:
The current formula in the `getPrice` function calculates the price of tokens in terms of ETH but does so incorrectly, leading to potential mispricing.

#### Description:
The formula used to calculate the price is:
```solidity
uint256 price = (ethBalance * (10 ** saleToken.decimals())) / saleTokenBalance;
```
This calculates the price as ETH per token instead of tokens per ETH, which is the intended functionality.

#### Impact:
This mispricing can severely affect the buying and selling of tokens. For example, with 1 ETH and 1000 tokens, the calculated price would be:
```plaintext
price = 1 * 10^18 / 1000 = 10^15 wei per token
```
This is incorrect, as it should reflect how many tokens can be bought for 1 ETH.

#### Mitigation:
To fix this issue, the formula should be inverted to calculate the price as tokens per ETH:
```solidity
uint256 price = (saleTokenBalance * (10 ** 18)) / ethBalance;
```

-----
-----
-----

### M-1 : **Unsupported Token Types in `buy` Function**

#### Summary:
The `buy` function does not account for various token types that may have specific properties or behaviors, which could lead to inaccuracies in token calculations and transfers.

#### Description:
The `buy` function currently assumes a fixed supply of tokens based on the ETH sent. However, if the `saleToken` has specific behaviors (e.g., tokens with transfer fees), this may lead to unexpected results during the transfer process. The function does not implement any checks or handling for such properties, which could result in users receiving incorrect amounts of tokens or transaction failures.

#### Impact:
Failure to account for different token types may lead to users receiving an incorrect number of tokens or facing unexpected transaction failures. This can result in a poor user experience and potential loss of funds.

#### Mitigation:
1. **Support for Various Token Types:**
   - Implement logic to ensure the `buy` function can accommodate different token properties by checking the characteristics of the `saleToken` before proceeding with the transfer.

2. **Use `safeTransfer`:**
   - Replace the `transfer` function with `safeTransfer` from the OpenZeppelin `SafeERC20` library to safely handle token transfers:
   ```solidity
   import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
   
   using SafeERC20 for ERC20;

   // Replace transfer with safeTransfer
   saleToken.safeTransfer(msg.sender, saleTokenAmount);
   ```
   This change will ensure that any potential issues during the transfer process are handled appropriately, reducing the risk of errors.


---
---

### M-2 : **No Allowance Check Before Token Transfer**

#### Summary:
The `sell` function does not perform an allowance check before transferring tokens from the user to the contract, potentially leading to unexpected transaction failures.

#### Description:
The `sell` function utilizes `transferFrom` to transfer `saleTokenAmount` tokens from the user to the contract. However, it does not check if the user has granted sufficient allowance for the contract to spend their tokens. If the user has not approved the contract for the `saleToken`, the `transferFrom` call will fail, leading to a revert and a poor user experience.

#### Impact:
The absence of an allowance check means that users may unknowingly attempt to sell tokens without granting the necessary approval, causing the function to revert and preventing them from completing the sale. This can lead to confusion and dissatisfaction, as users might not understand why their transaction failed.

#### Mitigation:
Before calling `transferFrom`, implement a check to ensure that the allowance is sufficient. You can do this by adding the following check before the transfer:
```solidity
if (saleToken.allowance(msg.sender, address(this)) < saleTokenAmount) {
    revert InsufficientAllowanceError();
}
```
This will inform users if they need to approve the contract for the specified `saleTokenAmount` before attempting to sell, improving the overall user experience.


--- 
Here’s the audit report for the `updateFairLaunchProperties` function, separated into distinct findings for access control and input validation:

---
---

### M-3: **Missing Input Validation in `updateFairLaunchProperties`**

#### Summary:
The `updateFairLaunchProperties` function does not validate the input parameters, which may lead to unexpected contract behavior.

#### Description:
The function assigns `_fairLaunchProperties` directly to the `fairLaunchProperties` variable without validating its contents. Given the critical nature of these properties, it is essential to implement checks to ensure that the values provided are valid and within acceptable ranges or formats.

#### Impact:
Invalid or erroneous values may be assigned to `fairLaunchProperties`, potentially resulting in incorrect contract behavior, leading to user confusion or loss of funds.

#### Mitigation:
Implement input validation checks before assigning the new properties. For example:
```solidity
require(_fairLaunchProperties.taxProperties.sellTax >= 0, "Sell tax must be non-negative");
require(_fairLaunchProperties.dateProperties.startDate < _fairLaunchProperties.dateProperties.endDate, "Start date must be before end date");
```
Incorporate these checks into the function:
```solidity
function updateFairLaunchProperties(FairLaunchProperties memory _fairLaunchProperties) external onlyOwner returns (bool) {
    require(_fairLaunchProperties.taxProperties.sellTax >= 0, "Sell tax must be non-negative");
    require(_fairLaunchProperties.dateProperties.startDate < _fairLaunchProperties.dateProperties.endDate, "Start date must be before end date");
    // Others requirements 

    fairLaunchProperties = _fairLaunchProperties;
    return true;
}
```

---
---

### M-4: **Missing Token Balance Check in `buy` Function**

#### Summary:
The `buy` function does not check if the contract has enough tokens available to fulfill the purchase, which may lead to transaction failures.

#### Description:
The `buy` function allows users to purchase tokens using ETH. However, it does not verify whether the contract has a sufficient balance of `saleToken` to accommodate the amount the user is attempting to buy. If the contract's `saleToken` balance is lower than the calculated `saleTokenAmount`, the `transfer` call will fail, resulting in a transaction revert and a poor user experience for the buyer.

#### Impact:
If the contract does not hold enough `saleToken`, the purchase will fail, causing users to lose their ETH without receiving the expected tokens. This can lead to frustration and a lack of trust in the contract's functionality.

#### Mitigation:
Add a check to ensure that the contract has enough `saleToken` available before proceeding with the transfer. This can be done by checking the balance of the `saleToken` in the contract and comparing it to the calculated `saleTokenAmount`. Here’s how you can implement it:
```solidity
// Check if the contract has enough saleToken to fulfill the purchase
require(saleToken.balanceOf(address(this)) >= saleTokenAmount, "Insufficient token balance in contract.");
```
Incorporate this check into the `buy` function as follows:
```solidity
function buy() public onlyNotBlacklisted(msg.sender) returns (uint saleTokenAmount) {
    if (!fairLaunchProperties.enabled) revert FairLaunchNotEnabledError();
    if (block.timestamp < fairLaunchProperties.dateProperties.startDate) revert FairLaunchNotStartedError();
    if (block.timestamp >= fairLaunchProperties.dateProperties.endDate) revert FairLaunchAlreadyEndedError();
   
    uint256 ethAmount = msg.value;
    if (address(msg.sender).balance < ethAmount) revert InsufficientETHBalanceError();

    uint256 price = getPrice();
    
    if (ethAmount == 0) revert InvalidAmountError();
    if (ethAmount < price) revert AmountMustBeGreaterThanPriceError();
    
    saleTokenAmount = (ethAmount * (10 ** saleToken.decimals())) / price;

    // Add token balance check here
    require(saleToken.balanceOf(address(this)) >= saleTokenAmount, "Insufficient token balance in contract.");
    
    bool success = saleToken.transfer(msg.sender, saleTokenAmount);
    if (!success) revert TransfersaleTokenFailed();

    return saleTokenAmount;
}
```

---
---

### M-5: **Implement Slippage Protection in `buy` and `sell` Functions**

#### Summary:
Introduce slippage protection to the `buy` and `sell` functions to safeguard users against unfavorable price changes during transactions.

#### Description:
Slippage occurs when the actual execution price of a trade differs from the expected price due to market fluctuations. By implementing slippage protection, users can specify a maximum acceptable slippage percentage, ensuring that they do not receive less than a defined minimum amount of tokens or ETH.

**For the `buy` function**:
- Calculate the minimum acceptable amount of tokens based on the specified slippage.
- Verify that the executed saleTokenAmount is greater than or equal to this minimum.

**For the `sell` function**:
- Similar checks should be applied to ensure that the ETH received meets the user's slippage tolerance.

#### Impact:
This implementation protects users from significant losses due to sudden market movements, thereby enhancing the trust and reliability of the contract.

#### Mitigation:
Add a `slippage` parameter to both functions and perform checks.


---
---
---


#### L-1: **Precision Loss in Price Calculation of `getPrice`**

#### Summary:
The division operation in the `getPrice` function can lead to significant precision loss, impacting the accuracy of the token pricing.

#### Description:
The formula used in the `getPrice` function involves a division operation:
```solidity
uint256 price = (ethBalance * (10 ** saleToken.decimals())) / saleTokenBalance;
```
When the sale token balance is small, this can lead to significant precision loss. 

#### Impact:
Precision loss can result in inaccurate pricing, favoring either buyers or sellers unfairly. For instance, if `ethBalance` is 1 wei and `saleTokenBalance` is 3 tokens, the price calculated would be:
```plaintext
price = 1 * 10^18 / 3 = 333333333333333333
```
This loss of precision can lead to users either overpaying or under-receiving tokens.

#### Mitigation:
To mitigate precision loss, consider using a safeMath library 

---
---

### L-2: **Missing Events in `buy` and `sell` Functions**

#### Summary:
The `buy` and `sell` functions lack events, preventing transaction tracking.

#### Description:
Events are not emitted for purchases and sales, making it hard to monitor on-chain activity.

#### Impact:
- Reduced transparency and tracking.
- Off-chain systems cannot detect transactions.

#### Mitigation:
Emit events in both functions to log transactions.

**For `buy`:**
```solidity
event TokensPurchased(address indexed buyer, uint256 ethAmount, uint256 saleTokenAmount);
```

**For `sell`:**
```solidity
event TokensSold(address indexed seller, uint256 saleTokenAmount, uint256 ethAmount);
```
