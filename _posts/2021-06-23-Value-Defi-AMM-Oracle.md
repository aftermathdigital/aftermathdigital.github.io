---
layout:		post
title:		Value DeFi MultiStablesVault Exploit
date:		2021-6-23 10:00:00
summary:	A deep-dive into the vulnerable oracle implementation which lost Value DeFi almost 9 million 3Crv.
categories:	exploit
---

In November 2020, Value DeFi was exploited and saw 8.9 million 3Crv drained from its MultiStablesVault, which was sold for 7.4 million DAI. As is often the case with multi-stable-vault exploitation, it was an unprotected pricing oracle which was vulnerable to a flash-loan attack which enabled the exploiters to perform their digital heist.

In a manner very similar to the [Harvest-Finance](https://aftermath.digital/exploit/2021/05/06/Harvest-Finance-Economic-Flash-Loan-Attack-October-2020/) flash-loan attack of October 2020, the vault contract was tricked into overvaluing the vault LP token's value with respect to a curve token - [3Crv](https://etherscan.io/address/0x6c3F90f043a72FA612cbac8115EE7e52BDe6E490) in this case.

In this post, I'll be looking at the implementation of Value Defi's pricing oracle, and the interaction between contracts which made all of this possible.

### Notable Contracts and Transactions:

- Exploiting Transaction: [0x46a03488247425f845e444b9c10b52ba3c14927c687d38287c0faddc7471150a](https://etherscan.io/tx/0x46a03488247425f845e444b9c10b52ba3c14927c687d38287c0faddc7471150a)
- (Value)MultiVaultStableProxy: [0x55BF8304C78Ba6fe47fd251F37d7beb485f86d26](https://etherscan.io/address/0x55bf8304c78ba6fe47fd251f37d7beb485f86d26)
- (Value)MultiStablesVault: [0xDdD7df28B1Fb668B77860B473aF819b03DB61101](https://etherscan.io/address/0xddd7df28b1fb668b77860b473af819b03db61101)
- (Value)MultiVaultBank: [0x8764f2c305b79680CfCc3398a96aedeA9260f7ff](https://etherscan.io/address/0x8764f2c305b79680cfcc3398a96aedea9260f7ff)
- (Value)ShareConverter: [0x57CDa125d0c7b146A8320614ccd6c55999d15BF2](https://etherscan.io/address/0x57cda125d0c7b146a8320614ccd6c55999d15bf2)
- (Value)StableSwap3PoolConverter: [0x8c2F33B3a580BAeb2A1F2D34bCC76E020a54338d](https://etherscan.io/address/0x8c2f33b3a580baeb2a1f2d34bcc76e020a54338d)
- (Value)MultiStablesVaultController: [0xba5D28F4ECEE5586D616024c74E4d791E01aDEE7](https://etherscan.io/address/0xba5d28f4ecee5586d616024c74e4d791e01adee7)
- Curve3Pool / StableSwap3Pool: [0xbEbc44782C7dB0a1A60Cb6fe97d0b483032FF1C7](https://etherscan.io/address/0xbebc44782c7db0a1a60cb6fe97d0b483032ff1c7)
- 3Crv token: [0x6c3F90f043a72FA612cbac8115EE7e52BDe6E490](https://etherscan.io/address/0x6c3f90f043a72fa612cbac8115ee7e52bde6e490)

### Deposit workflow

Firstly, the attacker deposited 25 million DAI into Value DeFi' MultiValueBank in exchange for just short of 25 million LP.

The `deposit` function exposed by the MultiVaultBank is as follows:

```c
//MultiVaultBank: [0x8764f2c305b79680CfCc3398a96aedeA9260f7ff]
function deposit(
    IValueMultiVault _vault,
    address _input,
    uint _amount,
    uint _min_mint_amount,
    bool _isStake,
    uint8 _flag) public discountCHI(_flag) {
    require(_vault.accept(_input), "vault does not accept this asset");
    require(_amount > 0, "!_amount");

    if (!_isStake) {
        _vault.depositFor(msg.sender, msg.sender, _input, _amount, _min_mint_amount); //[0]
    } else {
        uint _mint_amount = _vault.depositFor(msg.sender,
                                              address(this),
                                              _input,
                                              _amount,
                                              _min_mint_amount);
        
        _stakeVaultShares(address(_vault), _mint_amount);
    }
}
```

The `_input` parameter was the [DAI stablecoin](https://etherscan.io/token/0x6b175474e89094c44da98b954eedeac495271d0f) contract, and the `_vault` parameter passed to deposit was the [MultiStablesVault](https://etherscan.io/address/0xddd7df28b1fb668b77860b473af819b03db61101), who's deposit function `[0]`, looks like the following:

```c
//MultiStablesVault: [0xDdD7df28B1Fb668B77860B473aF819b03DB61101]
function depositFor(
    address _account,
    address _to,
    address _input,
    uint _amount,
    uint _min_mint_amount) public override checkContract returns (uint _mint_amount) {
    require(msg.sender == _account || msg.sender == vaultMaster.bank(address(this)),
            "!bank && !yourself");
    uint _pool = balance();
    require(totalDepositCap == 0 || _pool <= totalDepositCap, ">totalDepositCap");
    uint _before = 0;
    uint _after = 0;
    address _want = address(0);
    address _ctrlWant = IMultiVaultController(controller).want();
    if (_input == address(basedToken) || _input == _ctrlWant) {
        _want = _want;
        _before = IERC20(_input).balanceOf(address(this));
        basedToken.safeTransferFrom(_account, address(this), _amount);
        _after = IERC20(_input).balanceOf(address(this));
        _amount = _after.sub(_before); // additional check for deflationary tokens
    } else { //[0]
        _want = input2Want[_input];
        if (_want == address(0)) {
            _want = _ctrlWant;
        }
        IMultiVaultConverter _converter = converters[_want]; //[1]
        require(_converter.convert_rate(_input, _want, _amount) > 0, "rate=0");
        _before = IERC20(_want).balanceOf(address(this));
        IERC20(_input).safeTransferFrom(_account, address(_converter), _amount);
        _converter.convert(_input, _want, _amount); //[2]
        _after = IERC20(_want).balanceOf(address(this));
        _amount = _after.sub(_before); // additional check for deflationary tokens
    }
    require(_amount > 0, "no _want");
    _mint_amount = _deposit(_to, _pool, _amount, _want); //[3]
    require(_mint_amount >= _min_mint_amount, "slippage");
    }
```

In this case `address(basedToken)` and `_ctrlWant` are both [3Crv](https://etherscan.io/address/0x6c3f90f043a72fa612cbac8115ee7e52bde6e490), which isn't the input token specified by the attacker (which was DAI). This takes us to the `else` branch at `[0]`. Looking at `input2Want`, we see that when called with DAI as the input, the `_want` address is null - which results in`_want` being set to `_ctrlWant` (3Crv) after-all.

And so, after a roundabout trip, we end up at the line `IMultiVaultConverter _converter = converters[_want];`, at `[1]` with `_want` set to the address of the 3Crv token, which resolves a MultiVaultConverter for the 3Crv token. The purpose of this converter is to exchange the attacker's DAI for 3Crv. `converters[_want]` resolves to the address [0x8c2F33B3a580BAeb2A1F2D34bCC76E020a54338d](https://etherscan.io/address/0x8c2F33B3a580BAeb2A1F2D34bCC76E020a54338d), which is the StableSwap3PoolConverter. And the `convert` function called at `[2] `is the following:

```c
//StableSwap3PoolConverter: [0x8c2F33B3a580BAeb2A1F2D34bCC76E020a54338d]
function convert(
    address _input,
    address _output,
    uint _inputAmount) external override returns (uint _outputAmount) {
    require(
        vaultMaster.isVault(msg.sender) ||
        vaultMaster.isController(msg.sender) ||
        msg.sender == governance, "!(governance||vault||controller)");
    
    if (_inputAmount == 0) return 0;
    if (_output == address(token3Crv)) { // convert to 3Crv
        uint[3] memory amounts;
        for (uint8 i = 0; i < 3; i++) {
            if (_input == address(pool3CrvTokens[i])) {
                amounts[i] = _inputAmount;
                uint _before = token3Crv.balanceOf(address(this));
                stableSwap3Pool.add_liquidity(amounts, 1); //[0]
                uint _after = token3Crv.balanceOf(address(this));
                _outputAmount = _after.sub(_before);
                token3Crv.safeTransfer(msg.sender, _outputAmount);
                return _outputAmount;
            }
        }
    /* code below snipped for brevity*/
}
```

This is a big function, so I've omitted the lines which concern tokens other than 3Crv, since that's what we're interested in.

- The for loop is iterating over the native tokens supported by the Curve's stableSwap3Pool, which are DAI, USDC, and USDT respectively.
  - Given DAI is at index zero, the first iteration of the loop will result in the DAI the attacker deposited in Value DeFi's MultiStableVault being deposited into [Curve's pool](https://etherscan.io/address/0xbebc44782c7db0a1a60cb6fe97d0b483032ff1c7) via the `stableSwap3Pool.add_liquidity(amounts, 1);` at `[0]`.
  - The [stableSwap3Pool](https://etherscan.io/address/0xbebc44782c7db0a1a60cb6fe97d0b483032ff1c7) `add_liquidity` function is responsible for exchanging an asset for 3Crv.
  - Getting into the weeds of 3Crv is beyond the scope of this post, but you can see how the function works [here](https://etherscan.io/address/0xbebc44782c7db0a1a60cb6fe97d0b483032ff1c7#code); suffice to say DAI is deposited, and 3Crv (the pool's LP token) is returned at the rate deemed appropriate by Curve's algorithm.

- This iteration of the for loop ends in `return _outputAmount`, which lets the caller know how many 3Crv tokens the DAI was exchanged for.
  - Interestingly, the caller seems to ignore the return value and calculate the amount of tokens returned by checking the contract balance before and after the call to `convert`.

Returning to `depositFor` (the figure previous to the one above), the next step is to take these new 3Crv tokens, and deposit them in the `MultiStablesVault` using the `_deposit` function at `[3]`:

```c
//MultiStablesVault: [0xDdD7df28B1Fb668B77860B473aF819b03DB61101]
function _deposit(
    address _mintTo,
    uint _pool,
    uint _amount,
    address _want) internal returns (uint _shares) {
    uint _insuranceFee = vaultMaster.insuranceFee();
    if (_insuranceFee > 0) {
        uint _insurance = _amount.mul(_insuranceFee).div(10000);
        _amount = _amount.sub(_insurance);
        insurance = insurance.add(_insurance);
    }

    if (_want != address(basedToken)) {
        _amount = shareConverter.convert_shares_rate(_want, address(basedToken), _amount);
        if (_amount == 0) {
            // try [stables_2_basedWant] if [share_2_share] failed
            _amount = basedConverter.convert_rate(_want, address(basedToken), _amount);
        }
    }

    if (totalSupply() == 0) {
        _shares = _amount;
    } else {
        _shares = (_amount.mul(totalSupply())).div(_pool); //[0]
    }

    if (_shares > 0) {
        earn(_want); //[1]
        _mint(_mintTo, _shares); //[2]
    }
}
```

This function is fairly simple.

- There's an insurance fee taken, which we can ignore.
- There's some logic to determine how many 'shares' (LP Tokens) should be minted in exchange for the 3Crv tokens received in exchange for the attacker's deposited DAI.
- `_want` and `basedToken` are both 3Crv, so we fall straight through to the calculation for `_shares` at `[0]`:
  - `_shares = (_amount.mul(totalSupply())).div(_pool);`
  - This is the usual calculation to determine the value of a single 3Crv in pool LP, multiplied by the `_amount` of 3Crv being deposited.

- Finally, if `_shares` is greater than 0:
  - The `_earn` function at `[1]` puts this new 3Crv to work, as we'll see below.
  - The contract mints the attacker their share of the pool, given their initial DAI deposit at `[2]`.

```c
//MultiStablesVault: [0xDdD7df28B1Fb668B77860B473aF819b03DB61101]
function earn(address _want) public override {
    if (controller != address(0)) {
        IMultiVaultController _contrl = IMultiVaultController(controller);
        if (!_contrl.investDisabled(_want)) {
            uint _bal = available(_want);
            if ((_bal > 0) && (_want != address(basedToken) || _bal >= earnLowerlimit)) {
                IERC20(_want).safeTransfer(controller, _bal); //[0]
                _contrl.earn(_want, _bal);
            }
        }
    }
}
```

This contract (MultiStablesVault) is not the contract which invests the 3Crv in a strategy to farm yield - it's done by a [controller](https://etherscan.io/address/0xba5d28f4ecee5586d616024c74e4d791e01adee7) contract. We'll re-visit this controller contract later, as its role in this exploit is significant.

We can see the 3Crv tokens transferred from this contract to the [MultiStablesVaultController](https://etherscan.io/address/0xba5d28f4ecee5586d616024c74e4d791e01adee7) at `[0]`.

That concludes the deposit workflow for this contract: DAI is deposited, that DAI is added as liquidity to Curve's StableSwap3Pool in exchange for 3Crv, some amount of MultiStablesVault LP is minted and sent to the user in exchange for that 3Crv, and the 3Crv itself is sent to the controller contract to earn yield.

Here's a simplified, high-level diagram:

![deposit_workflow](/images/value_deposit.png)

### Withdrawal workflow

The attacker's withdrawal was asymmetric, they deposited DAI but withdrew 3Crv.

Withdrawal starts with the MultiVaultBank contract's `withdraw` function:

```c
//MultiVaultBank: [0x8764f2c305b79680CfCc3398a96aedeA9260f7ff]
function withdraw(
    address _vault,
    uint _shares,
    address _output,
    uint _min_output_amount,
    uint8 _flag) public discountCHI(_flag) {
    uint _userBal = IERC20(address(_vault)).balanceOf(msg.sender);
    if (_shares > _userBal) {
        uint _need = _shares.sub(_userBal);
        require(_need <= userInfo[_vault][msg.sender].amount, "_userBal+staked < _shares");
        unstake(_vault, _need, uint8(0));
    }
    IERC20(address(_vault)).safeTransferFrom(msg.sender, address(this), _shares);
    IValueMultiVault(_vault).withdrawFor(msg.sender, _shares, _output, _min_output_amount);//[0]
}
```

Similar to the `deposit` function, the `withdraw` function requires the address of a vault and an output token address. The attacker used the same `_vault` address ( [MultiStablesVault](https://etherscan.io/address/0xddd7df28b1fb668b77860b473af819b03db61101)) as they did when depositing, but they used 3Crv as the output token address.

There's a condition that will automatically un-stake some tokens to ensure the user has enough idle LP to make the withdrawal, if they don't have enough idle LP already.

The LP is optimistically transferred from the attacker and the vault's `withdrawFor` method is called at `[0]`:

```c
//MultiStablesVault: [0xDdD7df28B1Fb668B77860B473aF819b03DB61101]
function withdrawFor(
    address _account,
    uint _shares,
    address _output,
    uint _min_output_amount) public override returns (uint _output_amount) {
    //AFTERMATH: the balance_to_sell() calculation below is critical to the exploit.
    _output_amount = (balance_to_sell().mul(_shares)).div(totalSupply()); //[0]
    _burn(msg.sender, _shares);

    uint _withdrawalProtectionFee = vaultMaster.withdrawalProtectionFee();
    if (_withdrawalProtectionFee > 0) {
        uint _withdrawalProtection = _output_amount.mul(_withdrawalProtectionFee).div(10000);
        _output_amount = _output_amount.sub(_withdrawalProtection);
    }

    // Check balance
    uint b = basedToken.balanceOf(address(this));
    if (b < _output_amount) {
        uint _toWithdraw = _output_amount.sub(b);
        uint _wantBal = IMultiVaultController(controller).wantStrategyBalance(address(basedToken));
        if (_wantBal < _toWithdraw && allowWithdrawFromOtherWant[_output]) {
            // if balance is not enough and we allow withdrawing from other wants
			// AFTERMATH: CODE HERE OMITED FOR BREVITY
        }
        uint _withdrawFee = IMultiVaultController(controller).withdraw(address(basedToken),
                                                                       _toWithdraw);
        uint _after = basedToken.balanceOf(address(this));
        uint _diff = _after.sub(b);
        if (_diff < _toWithdraw) {
            _output_amount = b.add(_diff);
        }
        if (_withdrawFee > 0) {
            _output_amount = _output_amount.sub(_withdrawFee, "_output_amount < _withdrawFee");
        }
    }

    if (_output == address(basedToken)) {
        require(_output_amount >= _min_output_amount, "slippage");
        basedToken.safeTransfer(_account, _output_amount);
    } else {
        require(basedConverter.convert_rate(address(basedToken), _output, _output_amount) > 0,
                "rate=0");
        basedToken.safeTransfer(address(basedConverter), _output_amount);
        uint _outputAmount = basedConverter.convert(address(basedToken), _output, _output_amount);
        require(_outputAmount >= _min_output_amount, "slippage");
        IERC20(_output).safeTransfer(_account, _outputAmount);
    }
}
```

`_output_amount` is calculated from three values, `balance_to_sell()`, `_shares`, and `totalSupply` at `[0]`. If the attacker can maximise one of the multipliers, or minimize the divisor, then the `_output_amount` goes up. As you've probably guessed, the attacker can indeed manipulate one of these values in such a way...

```c
//MultiStablesVault: [0xDdD7df28B1Fb668B77860B473aF819b03DB61101]
function balance_to_sell() public view returns (uint) {
    uint bal = basedToken.balanceOf(address(this));
    if (controller != address(0)) bal = bal.add(
        IMultiVaultController(controller).balanceOf(address(basedToken), true));
    return bal.sub(insurance);
}
```

This function looks fairly straight forward, but it turns out `IMultiVaultController`'s `balanceOf` function is pretty wild compared to a normal ERC20 `balanceOf` function.

```c
// MultiStablesVaultController: [0xba5D28F4ECEE5586D616024c74E4d791E01aDEE7]
function balanceOf(address _want, bool _sell) external view returns (uint _totalBal) {
    uint _wlength = wantTokens.length;
    if (_wlength == 0) {
        return 0;
    }
    _totalBal = 0;
    for (uint i = 0; i < _wlength; i++) {
        address wt = wantTokens[i];
        uint _bal = wantStrategyBalance(wt);
        if (wt != _want) {
            _bal = shareConverter.convert_shares_rate(wt, _want, _bal); //[0]
            if (_sell) {
                _bal = _bal.mul(9998).div(10000); // minus 0.02% for selling
            }
        }
        _totalBal = _totalBal.add(_bal);
    }
}
```

This for loop is iterating over all of the tokens the `MultiVaultController` contract has ownership over (its `_want` tokens).

In the line marked `[0]`, the contract is checking how much each of the tokens it owns is worth when converted to the token the attacker is trying to withdraw, as below:

```c
// MultiStablesVaultController: [0xba5D28F4ECEE5586D616024c74E4d791E01aDEE7]    
function convert_shares_rate(
    address _input,
    address _output,
    uint _inputAmount) external override view returns (uint _outputAmount) {
    if (_output == address(token3CRV)) {
        if (_input == address(tokenBCrv)) { // convert from BCrv -> 3CRV
            uint[3] memory _amounts;
            _amounts[1] = depositBUSD.calc_withdraw_one_coin(_inputAmount, 1); // BCrv -> USDC
            _outputAmount = stableSwap3Pool.calc_token_amount(_amounts, true); // USDC -> 3CRV
        } else if (_input == address(tokenSCrv)) { // convert from SCrv -> 3CRV
            uint[3] memory _amounts;
            _amounts[1] = depositSUSD.calc_withdraw_one_coin(_inputAmount, 1); // SCrv -> USDC
            _outputAmount = stableSwap3Pool.calc_token_amount(_amounts, true); // USDC -> 3CRV
        } else if (_input == address(tokenHCrv)) { // convert from HCrv -> 3CRV
            _outputAmount = stableSwapHUSD.calc_withdraw_one_coin(_inputAmount, 1); // HCrv -> 3CRV
        } else if (_input == address(tokenCCrv)) { // convert from CCrv -> 3CRV
            uint[3] memory _amounts;
            uint usdc = depositCompound.calc_withdraw_one_coin(_inputAmount, 1); // CCrv -> USDC
            _amounts[1] = usdc;//convert_usdc_to_cusdc(usdc); // TODO: to implement
            _outputAmount = stableSwap3Pool.calc_token_amount(_amounts, true); // USDC -> 3CRV
        }
    } else if (_output == address(tokenBCrv)) {
        //AFTERMATH: Code omitted for brevity
    } else if (_output == address(tokenSCrv)) {
        //AFTERMATH: Code omitted for brevity
    } else if (_output == address(tokenHCrv)) {
        //AFTERMATH: Code omitted for brevity
    } else if (_output == address(tokenCCrv)) {
        //AFTERMATH: Code omitted for brevity
    }
    if (_outputAmount > 0) {
        uint _slippage = _outputAmount.mul(vaultMaster.convertSlippage(_input, _output)).div(10000);
        _outputAmount = _outputAmount.sub(_slippage);
    }
}
```

**This is absolutely massive**.

In order to calculate the amount of 3Crv available to withdraw, the contract is going to check how much 3Crv it can get if it sells its other Crv tokens for USDC and uses that USDC to purchase 3Crv.

**This is a naïve pricing oracle, and it's one which can be trivially manipulated**.

- The way this MultiStablesVaultController works, is it is considering the LP to be a fraction of all of the assets it owns
  - And those assets can move in price independently of each other.
- This leads to a condition where, by use of a flash-loan or other market-manipulation, it's possible to create a massive opportunity for arbitrage.
- In addition, because the MultiStablesVaultController has a large existing balance of 3Crv which it won't need to sell on the market to facilitate the withdrawal.
  - This is important, it allows the attacker to withdraw funds based on a manipulated price, but not be exposed to the slippage cost of actually trading those funds.

By manipulating the stableSwap3Pool pool into thinking USDC is more valuable than its other assets (by decreasing the quantity of USDC in the Curve pool), the amount of 3Crv that can be purchased with these other Crv tokens will be inflated.

And that's how the `_output_amount` variable can be maximised in the below formula:

```c
_output_amount = (balance_to_sell().mul(_shares)).div(totalSupply());
```

So `balance_to_sell()` is happy to consider all of the Crv tokens in its strategies when converting MutliStableVault LP into the output token the attacker has requested.

How exactly does it convert those LP tokens?

Recall that the MultiStableVaults `withdrawFor` method contained the following withdrawal, where `_toWithdraw` has now been inflated via a flash-loan:

```c
uint _withdrawFee = IMultiVaultController(controller).withdraw(address(basedToken), _toWithdraw);
```

Who's implementation is the following:

```c
function withdraw(address _want, uint _amount) external returns (uint _withdrawFee) {
    require(msg.sender == address(vault), "!vault");
    _withdrawFee = 0;
    uint _toWithdraw = _amount;
    uint _wantStrategyLength = wantStrategyLength[_want];
    uint _received;
    for (uint _sid = _wantStrategyLength; _sid > 0; _sid--) {
        StrategyInfo storage sinfo = strategies[_want][_sid - 1];
        IMultiVaultStrategy _strategy = IMultiVaultStrategy(sinfo.strategy);
        uint _stratBal = _strategy.balanceOf();
        if (_toWithdraw < _stratBal) {
            _received = _strategy.withdraw(_toWithdraw);
            _withdrawFee = _withdrawFee.add(_strategy.withdrawFee(_received));
            return _withdrawFee;
        }
        _received = _strategy.withdrawAll(); //[0]
        _withdrawFee = _withdrawFee.add(_strategy.withdrawFee(_received));
        if (_received >= _toWithdraw) {
            return _withdrawFee;
        }
        _toWithdraw = _toWithdraw.sub(_received); //[1]
    }
    if (_toWithdraw > 0) {
        // still not enough, try to withdraw from other wants strategies
        // AFTERMATH: omitted for brevity
    }
    return _withdrawFee;
}
```

- The for loop is iterating over all the strategies that use the `_want` token as their native token.
  - It withdraws `_want` from a strategy at `[0]`.
  - It decrements the amount of `_want` it needs to get from other strategies at `[1]`.
- Before the attacker deposited their DAI for LP, the quantity of 3Crv in these strategies was 8.9 million 3Crv.
- By depositing 25 million DAI, an additional 24.95 million 3Crv was added to that balance.
- That leaves approximately 34 million 3Crv tokens in this vault - so the attacker is able to drain that many tokens before being exposed to the slippage that would be experienced from actually trading other Crv tokens for 3Crv - which is what they did.


And here's another simplified diagram:

![withdraw_workflow](/images/value_withdraw.png)

## Conclusion

It's been subtly mis-reported that this exploit stole 7.4 million DAI from Value DeFi. As we can see, what the attacker really did was drain 8.9 million 3Crv from Value DeFi's MultiStableVaultController, which they then sold for 7.4 million DAI.

This is a classic case of a flash-loan manipulated pricing oracle.

The oracle simply used the current rate of exchange between various Crv tokens and the 3Crv token, using USDC as an intermediary. By manipulating the exchange rate of USDC and 3Crv, the oracle massively over-estimated how much 3Crv should be purchased.

Here's the output of contract which exploits this functionality:

![exploit](/images/valuedefi_exploit.png)
