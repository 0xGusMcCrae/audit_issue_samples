## Impact

In payment.sol, `sweepToken` and `refundETH` are used to retrieve tokens sent to the contract. Neither has any restriction on who can call them to get tokens/ETH out, so anyone can claim an inheriting contract's balances. 

https://github.com/code-423n4/2023-01-numoen/blob/2ad9a73d793ea23a25a381faadc86ae0c8cb5913/src/periphery/Payment.sol#L35-L46

    function sweepToken(address token, uint256 amountMinimum, address recipient) public payable {
        uint256 balanceToken = Balance.balance(token);
        if (balanceToken < amountMinimum) revert InsufficientOutputError();

        if (balanceToken > 0) {
            SafeTransferLib.safeTransfer(token, recipient, balanceToken);
        }
    }

    function refundETH() external payable {
        if (address(this).balance > 0) SafeTransferLib.safeTransferETH(msg.sender, address(this).balance);
    }

LiqudityManager and LendgineRouter both inherit these functions. Generally neither will hold a balance, but in `LiquidityManager.collect`, `LiquidityManager.removeLiquidity`, and `LendgineRouter.burn`, if `params.recipient` is `address(0)`, it will use `address(this)` as the recipient for the `collect`/`withdraw` call to the pair or `burn` call to the lendgine. When these tokens arrive in the LiquidityManager/LendgineRouter contract, they can be stolen by anyone and would be an easy target for a frontrunning bot (or anyone monitoring the contract's balances).

https://github.com/code-423n4/2023-01-numoen/blob/2ad9a73d793ea23a25a381faadc86ae0c8cb5913/src/periphery/LendgineRouter.sol#L262

https://github.com/code-423n4/2023-01-numoen/blob/2ad9a73d793ea23a25a381faadc86ae0c8cb5913/src/periphery/LiquidityManager.sol#L206

https://github.com/code-423n4/2023-01-numoen/blob/2ad9a73d793ea23a25a381faadc86ae0c8cb5913/src/periphery/LiquidityManager.sol#L233

## Proof of Concept

In LendgineRouterTest.t.sol, edit testBurnNoRecipient() to include the following code at the end (after line 717):

https://github.com/code-423n4/2023-01-numoen/blob/2ad9a73d793ea23a25a381faadc86ae0c8cb5913/test/LendgineRouterTest.t.sol#L717

    /////////////////////////////////////////
    ////////////////// POC //////////////////
    /////////////////////////////////////////
    
    uint256 balanceToSteal = token1.balanceOf(address(lendgineRouter));

    vm.prank(dennis);
    lendgineRouter.sweepToken(
      address(token1),
      balanceToSteal,
      dennis
    );

    assertEq(0, token1.balanceOf(address(lendgineRouter)));
    assertEq(balanceToSteal, token1.balanceOf(dennis));

`dennis` is a standin for any random account

This shows that if there is any balance in the LedgineRouter, it can be stolen by anyone calling sweepToken.

## Tools Used

Manual analysis, foundry

## Recommended Mitigation 

Track individual users' balances so that only the correct owner of the assets can withdraw them. For example, in `collect`, `collectAmount` could be stored as the user's eligible balance if the recipient is `address(this)`

or 

Restrict access to `sweepToken` and `refundETH` and have a trusted admin handle their use (although that adds its own centralization risks - the admin themselves could steal the funds).

or

Simply revert if `param.recipient` is `address(0)` so that tokens/eth don't arrive in the LendgineRouter or LiquidityManager in the first place.
