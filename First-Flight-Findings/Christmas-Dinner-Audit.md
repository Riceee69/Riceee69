### My Findings of the [2024-12-Christmas-Dinner](https://codehawks.cyfrin.io/c/2024-12-christmas-dinner)
# Christmas Dinner - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Improper nonReentrant logic](#H-01)
    - ### [H-02. withdraw() function logic needs to withdraw the Ether amount of the contract as well.](#H-02)
    - ### [H-03. withdraw() needs to add a time access check](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Improved refund() functionality](#M-01)
    - ### [M-02. setDeadline() needs 0 check and to update deadlineSet](#M-02)
    - ### [M-03. receive() logic should be same as deposit()](#M-03) (covers 2 mediums)
- ## Low Risk Findings
    - ### [L-01. changeParticipationStatus() needs a participant check](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #31

### Dates: Dec 19th, 2024 - Dec 26th, 2024

[View Full Report Here](https://codehawks.cyfrin.io/c/2024-12-christmas-dinner/results?lt=contest&sc=reward&sj=reward&page=1&t=report)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 4
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Improper nonReentrant logic            



## Summary

**The `nonReentrant` modifier is not implemented properly, the **`locked `**variable is not updated to **`true `**inside the modifier. Allowing recursive calls to the function.** 

## Vulnerability Details

**The `nonReentrant` modifier isn't implemented properly and doesn't prevent reentrant calls, the function is vulnerable to reentrancy attacks, which allows an attacker to repeatedly call the `refund()` function before the state updates to prevent further withdrawals.**

### Attack Scenario

Here’s how an attacker might exploit the reentrancy vulnerability for ETH:

1. **First Call to `refund`:**

   * The attacker calls `refund` and enters `_refundETH`.
   * ETH is sent to the attacker’s contract via `_to.transfer(refundValue)`.
   * The attacker’s fallback/receive function is triggered.
2. **Reentrant Call:**

   * In the fallback function, the attacker calls `refund` again.
   * Since `locked `hasn't been updated the modifier doesn't prevent reentrancy.
   * Because `etherBalance[_to]` has not yet been set to 0, the attacker can withdraw ETH again.
3. **Repeat:**

   * The attacker repeats this process until all ETH in the contract is drained

ERC20 tokens are safe because they are immediately updated to 0 once they are transferred in`_refundERC20`

## Impact

The enitre ether of the contract can be drained by a bad actor, by repeatedly calling the `refund()` function using a `fallback()`/`receive() `**function call before the `etherBalance `mapping is updated.**

## Tools Used

**Manual Review**

## Recommendations

**Update the nonReentrant modifier to ensure it works as intended.**\
&#x20;  &#x20;

```Solidity
modifier nonReentrant() {
        require(!locked, "No re-entrancy");
        locked = true;//added
        _;
        locked = false;
    }
```

## <a id='H-02'></a>H-02. withdraw() function logic needs to withdraw the Ether amount of the contract as well.            



## Summary

The `withdraw()` function only withdraws the ERC20 tokens from the contract to the `host`. It needs to also withdraw the Ether deposited by users as funds for the Christmas dinner.

## Impact

Since there's no other way to withdraw the ether to the host, The ether in the contract is kinda 'stuck' for the host. So they don't get the entirity of the funds for the Chritmas Dinner. 

```Solidity
function withdraw() external onlyHost {
    address _host = getHost();
    i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
    i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
    i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
}
```



## Tools Used

Manual Review

## Recommendations

Make the host wallet address payable and send the contract ether as well.

```solidity
    function withdraw() external onlyHost {
        address payable _host = payable(getHost());//added (made the address payable)
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));

        //need to send ether balance too(added)
        (bool success, ) = _host.call{value: address(this).balance}("");
        require(success, "Ether transfer failed");
    }
```


## <a id='H-03'></a>H-03. withdraw() needs to add a time access check            



## Summary

**`withdraw()` needs to check if the function is getting called before or after the deadline, to ensure the host can only withdraw the funds after the deadline is over. **

## Impact

Without any time access check, a malicious host can remove all the funds from the contract with a single function call before the deadline is over. Resulting in loss of user funds and a breach to the protocol logic. 

## Tools Used

Manual Review

## Recommendations 

Add a check at the beginning of the `withdraw() `method, to ensure that the function can only be accessed after the deadline has crossed.

&#x20;

```Solidity
require(block.timestamp > deadline, "Withdrawal not allowed before deadline");
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Improved refund() functionality            



# Summary

The `participant `mapping should be updated inside `refund()` function to prevent a replay attack.

## Vulnerability Details

If the `participant[msg.sender]` flag is not set to `false` after the first refund, the same participant can call the `refund` function again. While the `_refundERC20 `and `_refundETH `functions won’t transfer additional funds due to balances being zero, the `emit Refunded(msg.sender)` line will still execute. As well as if he wants to deposit funds again it will emit `GenerousAdditionalContribution`inplace of `NewSignup`

## Impact

This will result is a series of false emits being emitted by the contract leading to confusions/misleading information to the protocol users/observers that someone refunded multiple times and/or fake additional contributions. 

## Tools Used

Manual Review

## Recommendations

Update the `participant[msg.sender]` to `false `in the `refund()` function. And add a check at the beginning to check if the msg.sender is a participant or not(This check has been already mentioned in the given `report.md`)


## <a id='M-02'></a>M-02. setDeadline() needs 0 check and to update deadlineSet            



## Summary

The `seatDeadline()` doesn't update the `deadlineSet`to `true `when the deadline is set and neither has any 0 check implemented as a safety check. 

## Vulnerability Details

```Solidity
function setDeadline(uint256 _days) external onlyHost {
    if(deadlineSet) {
        revert DeadlineAlreadySet();
    } else {
        deadline = block.timestamp + _days * 1 days;
        emit DeadlineSet(deadline);
    }
}
```

If we dont check whether `_days`is 0 or not, it could lead to a Denial of Service. And if the `deadlineSet `isn't updated to `true `when the `deadline `is set, the host can keep changing the `deadline `whenever they wish to.  The `revert DeadlineAlreadySet()` will never be executed.

## Impact

Attack:\
After the contract has been deployed with a `deadline `set, a malicious host can call the function later again with `_days = 0`then the deadline would become the current `block.timestamp`, meaning the deadline is effectively immediate. Any functionality dependent on this deadline would become inaccessible as soon as the transaction is mined, as subsequent blocks would have a `block.timestamp` greater than `deadline`. And then the host could withdraw everything out scamming the participants. 

This could serve as a DoS (Denial of Service). Resulting in users not being able to access their funds anymore.

A malicious host can also change the deadline suddenly to an earlier date without prior notice, resulting in participants not able to refund their funds if they planned to later because the `deadlineSet `value is never updated

## Tools Used

Manual Review 

## Recommendations

Add a check for` _days > 0`and `deadlineSet = true`in the else block so the state is updated when `deadline `is set

```Solidity
function setDeadline(uint256 _days) external onlyHost {
        require(_days > 0, "Deadline must be in the future");//added
        if(deadlineSet) {
            revert DeadlineAlreadySet();
        } else {
            deadlineSet = true;//added
            deadline = block.timestamp + _days * 1 days;
            emit DeadlineSet(deadline);
        }
    }
```

## <a id='M-03'></a>M-03. receive() logic should be same as deposit()            



## Summary

receive() should check and set participant status and emit proper events. 

## Vulnerability Details

This function doesn't check whether the sender is already a participant or not, neither updates the `participant `status on signup Resulting in always NewSignup event emission but no record in storage which is misleading. It also allows a user to keep depositing ether even after the deadline is over, resulting in wrong deposit logic.

```Solidity
receive() external payable {
    etherBalance[msg.sender] += msg.value;
    emit NewSignup(msg.sender, msg.value, true);
}
```

## Impact

User, although being an active participant loses his privelege to access participant only functions. Which could potentially lead to inaccessible funds on the user's end.

## Tools Used

Manual Review

## Recommendations 

add `if-else` conditon to check for old/new participant as well as update participation status and deadline modifier (same as deposit logic)

```Solidity
    receive() external payable beforeDeadline {
        //Implemented proper logic as deposit
        if(participant[msg.sender]) {
            etherBalance[msg.sender] += msg.value;
            emit GenerousAdditionalContribution(msg.sender, msg.value);
        }else{
            participant[msg.sender] = true;//added
            etherBalance[msg.sender] += msg.value;
            emit NewSignup(msg.sender, msg.value, true);
        }
    }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. changeParticipationStatus() needs a participant check            



## Summary

The `changeParticipationStatus()`function doesn't check if the msg.sender is a registered participant of the protocol

## Vulnerability Details

Any random user can call the `changeParticipationStatus()`function, irrespective of them being a participant. This error allows a random user to be updated as the participant of the protocol if they call this function before the deadline. Now since the `participant[msg.sender] = true`for the non-participating user, resulting in a wrong state and on calling the `deposit()` function for the first time by them they emit the wrong event `GenerousAdditionalContribution`as well as they get acess to the `refund()`function given that refund has a participant check in place, resulting in wrong emits `Refunded`

## Impact

Improper use of event emissions can mislead users or off-chain systems, such as dApps or explorers. Emitting false or extra events for actions that didn't occur can deceive systems and trigger unintended behavior

## Tools Used

Manual Review

## Recommendations

To ensure an account is a participant it has to  have previously deposited tokens/ether into the contract and then changed their participation status. So we add that check initially to filter proper users who should be allowed to access this function.

```Solidity
    function changeParticipationStatus() external {
        //!!!Need to only allow prior particpiants to change their status
        require(_hasBalanceInContract(msg.sender), "Only participants can change their status");//added
        if(participant[msg.sender]) {
            participant[msg.sender] = false;
        } else if(!participant[msg.sender] && block.timestamp <= deadline) {
            participant[msg.sender] = true;
        } else {
            revert BeyondDeadline();
        }
        emit ChangedParticipation(msg.sender, participant[msg.sender]);
    }
```

and a `_hasBalanceInContract` is defined to check if the user has funds deposited in the contract:

```Solidity
    function _hasBalanceInContract(address _user) private view returns (bool) {
        uint256 totalBalance = balances[_user][address(i_USDC)] + 
      balances[_user][address(i_WBTC)] + balances[_user][address(i_WETH)] + etherBalance[_user];

        if(totalBalance > 0) {
            return true;
        }
        return false;
    }
```

# Informational(Not Considered for the audit)

## <a id='Informational'></a>Info. Added Re-SignUp events for better clarification            


## Summary

When a participant changes their participation status from true to false, and then deposits funds to the contract again, it will emit a `NewSignUp` event which is technically not true as the user had already Signed up previously and did not refund his funds. It's more like a Re-Signing up. 

## Impact

Incorrect or missing events can hinder off-chain monitoring and cause issues with integrations

## Tools Used

Manual Review

## Recommendations

add a new event such as:    &#x20;

```Solidity
event ReSignup(address indexed, uint256 indexed, bool indexed);//added for more clarity
```

And then change the emission logic of the `deposit()` and `receive()` functions for better clarification:   &#x20;

```Solidity
function deposit(address _token, uint256 _amount) external beforeDeadline {
        if(!whitelisted[_token]) {
            revert NotSupportedToken();
        }
        if(participant[msg.sender]){
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            emit GenerousAdditionalContribution(msg.sender, _amount);
        } else {
          //if-else logic added to emit proper events
            if(_hasBalanceInContract(msg.sender)){
                emit ReSignup(msg.sender, _amount, true);   
            }else{
                emit NewSignup(msg.sender, _amount, true);
            }
            participant[msg.sender] = true;
            balances[msg.sender][_token] += _amount;
            IERC20(_token).safeTransferFrom(msg.sender, address(this), _amount);
            //emit NewSignup(msg.sender, _amount, getParticipationStatus(msg.sender));
        }
    }


    receive() external payable {
        //Implemented proper logic as deposit
        if(participant[msg.sender]) {
            etherBalance[msg.sender] += msg.value;
            emit GenerousAdditionalContribution(msg.sender, msg.value);
        }else{
          //if-else logic added for proper event emissions
            if(_hasBalanceInContract(msg.sender)){
                emit ReSignup(msg.sender, msg.value, true);   
            }else{
                emit NewSignup(msg.sender, msg.value, true);
            }
            participant[msg.sender] = true;//added
            etherBalance[msg.sender] += msg.value;
            //emit NewSignup(msg.sender, msg.value, true);
        }
    }
```

So that now if a participant redeposits after changing their status to false it will emit a `ReSignup`event making the phenomenon clearer. 



