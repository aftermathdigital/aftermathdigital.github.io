---
layout:		post
title:		Technical Details of the Harvest Finance Flash Loan Attack	
date:		2021-05-26 12:00:00
summary:	A deep-dive into the economic exploit that hit Harvest Finance in October 2020.
categories:	exploit
---
In October 2020, Harvest Finance was hit by a large-scale economic attack, involving a flash loan and a pool-price-manipulation.

The exploiting contract syphoned approximately 11 million dollars-worth of stablecoin from Harvest Finance's vaults.

In this post, we're going to look at the underlying implementation of the vault, and explore what it was that made this attack possible.

### Notable Contracts

- Exploiting Transaction: [0x35f8d2f572fceaac9288e5d462117850ef2694786992a8c3f6d02612277b0877](https://etherscan.io/tx/0x35f8d2f572fceaac9288e5d462117850ef2694786992a8c3f6d02612277b0877)
- VaultProxy (fUSDC): [0xf0358e8c3CD5Fa238a29301d0bEa3D63A17bEdBE](https://etherscan.io/address/0xf0358e8c3cd5fa238a29301d0bea3d63a17bedbe)
- CRVStrategyStableMainnet: [0xD55aDA00494D96CE1029C201425249F9dFD216cc](https://etherscan.io/address/0xd55ada00494d96ce1029c201425249f9dfd216cc)
- VaultYCRV: [0xF2B223Eb3d2B382Ead8D85f3c1b7eF87c1D35f3A](https://etherscan.io/address/0xf2b223eb3d2b382ead8d85f3c1b7ef87c1d35f3a)
- CRVStrategyYCRVMainnet: [0x2427DA81376A0C0a0c654089a951887242D67C92](https://etherscan.io/address/0x2427da81376a0c0a0c654089a951887242d67c92)

### Workflow for token deposits and withdrawals

As we can see from the [exploiting transaction](https://etherscan.io/tx/0x35f8d2f572fceaac9288e5d462117850ef2694786992a8c3f6d02612277b0877), the interaction with Harvest Finance's infrastructure can be summed up as the `deposit` and `withdraw` functionality of the VaultProxy.

The attacker deposits 49,977,468.555526 USDC into the contract, and receives 51,456,280.788906 fUSDC shares in return.

The seller later deposits the 51,456,280.788906 fUSDC and receives 50,298,684.215219 USDC.

What's going on? Depositing 49.977 million USDC and immediately withdrawing 51.456 million USDC? Let's work it out.

Beneath the hood, this contract uses the curve.fi yPool - a multi-asset stablecoin pool where users can deposit DAI, USDT, USDC, or TUSD in exchange for yDAI, yUSDT, yUSDC, or yTUSD, which earn interest as the underlying token is used to provide liquidity to curve's AMM. In this case, Harvest Finance were using the yAsset tokens to purchase yCRV, which they could further invest in order to maximize yield.

### Depositing USDC

Some underlying asset (USDC) is sent to the VaultProxy, and the `deposit` function of the VaultYCRV function is `delegatecall`ed, which wraps the following `_deposit` function:

```c
function _deposit(uint256 amount, address sender, address beneficiary) internal{
  require(amount > 0, "Cannot deposit 0");
  require(beneficiary != address(0), "holder must be defined");

  if (address(strategy) != address(0)) {
    require(strategy.depositArbCheck(), "Too much arb");
  }

  /*
    todo: Potentially exploitable with a flashloan if
    strategy under-reports the value.
  */
  uint256 toMint = totalSupply() == 0
      ? amount
      : amount.mul(totalSupply()).div(underlyingBalanceWithInvestment());
  _mint(beneficiary, toMint);

  underlying.safeTransferFrom(sender, address(this), amount);

  // update the contribution amount for the beneficiary
  contributions[beneficiary] = contributions[beneficiary].add(amount);
  emit Deposit(beneficiary, amount);
}
```

*There's a pretty unfortunate comment in here... which I'll choose to ignore...*

Firstly, there's a requirement that `strategy.depositArbCheck()` returns true.

```c
function depositArbCheck() public view returns(bool) {
  uint256 currentPrice = underlyingValueFromYCrv(ycrvUnit);
  if (currentPrice < curvePriceCheckpoint) {
    return currentPrice.mul(100).div(curvePriceCheckpoint) > 100 - arbTolerance;
  } else {
    return currentPrice.mul(100).div(curvePriceCheckpoint) < 100 + arbTolerance;
  }
}
```
The purpose of this function is to check that the deviation in the price difference of yCRV and an underlying yAsset is less than at some known '`checkpoint`'. As it happens, the `arbTolerence` in this instance was set at three percent, a value far too high to protect the contract when it has enough liquidity to support millions of dollars of transacting without causing more than the pre-set three percent deviation. So this check can be passed, as it is overly permissive - however it's worth noting that a price deviation of three percent from the last checkpoint will cause the transaction to be reverted.

There's another critical function lurking in this check, which we can't skip over: `underlyingValueFromYCrv`

```c
/**
* Returns the value of yCRV in y-token (e.g., yCRV -> yDai) accounting for slippage and fees.
*/
function underlyingValueFromYCrv(uint256 ycrvBalance) public view returns (uint256) {
  return IPriceConvertor(convertor).yCrvToUnderlying(ycrvBalance, uint256(tokenIndex));
}
```

This function is responsible for getting the current market rate of yCRV to yAsset. It's worth noting that one of the ways the yCRV pool maintains the correct ratio of yAssets is through arbitrage opportunities between yCRV and the underlying yAssets when the ratios of those yAssets deviate.

Returning to the `_deposit` function, we now understand that the line `require(strategy.depositArbCheck(), "Too much arb");` is well intentioned, but needs to be using a much lower `arbTolerance` given the massive amount of liquidity in this pool, possibly with more frequent checkpoints.

Next up is the calculation behind `toMint`. We'll only consider the case where the `totalSupply != 0`.

This calculation is responsible for determining how much of the total assets locked up in the underlying vault will be purchased by the tokens which are being deposited. It is essentially minting LP in exchange for USDC.

The calculation is fairly simple:
* What is the total amount of LP available
* Divided by the total underlying balance of the vault
  * the amount of underlying represented by 1 LP - or the value of 1 LP in USDC.
* Multiplied by the amount of new underlying being deposited

There is a subtlety here which is not immediately clear - who's to say what the value of 1 LP is in USDC? It turns out, this is a difficult problem to solve, and one that is frequently solved insecurely.

Let's consider the implementation of `underlyingBalanceWithInvestment`, which represents the total balance of the vault in USDC.

```c
function underlyingBalanceWithInvestment() view public returns (uint256) {
  if (address(strategy) == address(0)) {
    // initial state, when not set
    return underlyingBalanceInVault();
  }
  return underlyingBalanceInVault().add(strategy.investedUnderlyingBalance());
}
```

It's made up of two variables:
- whatever is returned by `underlyingBalanceInVault()` (which is simply USDC)
- plus `strategy.investedUnderlyingBalance()`
  - which is the USDC value of the contract's investments, yCRV in this case.

The former just returns the amount of underlying USDC ERC20 owned by this contract.

The latter has to query the underlying strategy and gauge the equivalent underlying value of whatever is locked up in the strategy - or the USDC value of the yCRVthe strategy is farming.

Do you see where this is headed?

```c
/**
* Returns the underlying invested balance. This is the amount of yCRV that we are entitled to
* from the yCRV vault (based on the number of shares we currently have), converted to the
* underlying assets by the Curve protocol, plus the current balance of the underlying assets.
*/
function investedUnderlyingBalance() public view returns (uint256) {
  uint256 shares = IERC20(ycrvVault).balanceOf(address(this));
  uint256 price = IVault(ycrvVault).getPricePerFullShare();
  // the price is in yCRV units, because this is a yCRV vault
  // the multiplication doubles the number of decimals for shares, so we need to divide
  // the precision is always 10 ** 18 as the yCRV vault has 18 decimals
  uint256 precision = 10 ** 18;
  uint256 ycrvBalance = shares.mul(price).div(precision);
  // now we can convert the balance to the token amount
  uint256 ycrvValue = underlyingValueFromYCrv(ycrvBalance);
  return ycrvValue.add(IERC20(underlying).balanceOf(address(this)));
}
```

How does `underlyingValueFromYCrv(ycrvBalance)` work?

```c
/**
* Returns the value of yCRV in y-token (e.g., yCRV -> yDai) accounting for slippage and fees.
*/
function underlyingValueFromYCrv(uint256 ycrvBalance) public view returns (uint256) {
  return IPriceConvertor(convertor).yCrvToUnderlying(ycrvBalance, uint256(tokenIndex));
}
```

This is where things get weird. This converter contract is closed-source. A decompiler renders it as such

```python
def storage:
  unknown262d6152Address is addr at storage 0

def unknown262d6152() payable: 
  return unknown262d6152Address

def _fallback() payable: # default function
  revert

#keccak hash confirms this is: yCrvToUnderlying(uint256,uint256)
def unknown8c92b130(uint256 _param1, uint256 _param2) payable: 
  require calldata.size - 4 >= 64
  require ext_code.size(unknown262d6152Address)
  static call unknown262d6152Address.0xcc2b27d7 with:
          gas gas_remaining wei
         args _param1, ('signextend', 15, ('signextend', 15, ('param', '_param2')))
  if not ext_call.success:
      revert with ext_call.return_data[0 len return_data.size]
  require return_data.size >= 32
  return ext_call.return_data[0]
```

Fortunately, there is only one function, and only one storage slot in use.

The storage slot contained: [0xd6aD7a6750A7593E092a9B218d66C0A814a3436e](https://etherscan.io/address/0xd6aD7a6750A7593E092a9B218d66C0A814a3436e)

Of the functions in this contract, one has the `0xcc2b27d7` signature that is called in `unknown8c92b130`:

- `calc_withdraw_one_coin(uint256,int128)`
  - Because the first 4 bytes of `keccak-256('calc_withdraw_one_coin(uint256,int128)')`  are `0xcc2b27d7`

The function called here is a wrapper around

```python
@private
@constant
def _calc_withdraw_one_coin(_token_amount: uint256, i: int128, rates: uint256[N_COINS]) -> uint256:
    # First, need to calculate
    # * Get current D
    # * Solve Eqn against y_i for D - _token_amount
    crv: address = self.curve
    A: uint256 = Curve(crv).A()
    fee: uint256 = Curve(crv).fee() * N_COINS / (4 * (N_COINS - 1))
    fee += fee * FEE_IMPRECISION / FEE_DENOMINATOR  # Overcharge to account for imprecision
    precisions: uint256[N_COINS] = PRECISION_MUL
    total_supply: uint256 = ERC20(self.token).totalSupply()

    xp: uint256[N_COINS] = PRECISION_MUL
    S: uint256 = 0
    for j in range(N_COINS):
        xp[j] *= Curve(crv).balances(j)
        xp[j] = xp[j] * rates[j] / LENDING_PRECISION
        S += xp[j]

    D0: uint256 = self.get_D(A, xp)
    D1: uint256 = D0 - _token_amount * D0 / total_supply
    xp_reduced: uint256[N_COINS] = xp

    # xp = xp - fee * | xp * D1 / D0 - (xp - S * dD / D0 * (0, ... 1, ..0))|
    for j in range(N_COINS):
        dx_expected: uint256 = 0
        b_ideal: uint256 = xp[j] * D1 / D0
        b_expected: uint256 = xp[j]
        if j == i:
            b_expected -= S * (D0 - D1) / D0
        if b_ideal >= b_expected:
            dx_expected += (b_ideal - b_expected)
        else:
            dx_expected += (b_expected - b_ideal)
        xp_reduced[j] -= fee * dx_expected / FEE_DENOMINATOR

    dy: uint256 = xp_reduced[i] - self.get_y(A, i, xp_reduced, D1)
    dy = dy / precisions[i]

    return dy
```
I'm not going to dig through the maths of this contract - what it is is a Curve.fi contract calculating how much yAsset can be purchased by a single yCRV token, in the current pool.

**This is an absolute show-stopper.**

When this strategy invests the yAsset, it converts it into yCRV, which is the token it actually farms.

By creating a price imbalance between a yAsset and yCRV (say... through a flash-loan), the value returned by this function can be manipulated.

Let's say we're market manipulator dealing with yUSDC

1. We trade a massive yUSDC for yOtherAsset in the pool
2. This drives down the price of yUSDC
3.  We come by and deposit large amounts of USDC into the fUSDC vault
4. The term `underlyingBalanceWithInvestment()`  in the calculation `amount.mul(totalSupply()).div(underlyingBalanceWithInvestment());` reduced
5. Resulting in a larger value in toMint than we'd otherwise be entitled to.

But how does this result in profit when we go-on to withdraw from the pool?

### Withdrawing USDC

```c
function withdraw(uint256 numberOfShares) external {
  require(totalSupply() > 0, "Vault has no shares");
  require(numberOfShares > 0, "numberOfShares must be greater than 0");
  uint256 totalSupply = totalSupply();
  _burn(msg.sender, numberOfShares);

  uint256 underlyingAmountToWithdraw = underlyingBalanceWithInvestment()
      .mul(numberOfShares)
      .div(totalSupply);

  if (underlyingAmountToWithdraw > underlyingBalanceInVault()) {
    // withdraw everything from the strategy to accurately check the share value
    if (numberOfShares == totalSupply) {
      strategy.withdrawAllToVault();
    } else {
      uint256 missing = underlyingAmountToWithdraw.sub(underlyingBalanceInVault());
      strategy.withdrawToVault(missing);
    }
    // recalculate to improve accuracy
    underlyingAmountToWithdraw = Math.min(underlyingBalanceWithInvestment()
        .mul(numberOfShares)
        .div(totalSupply), underlyingBalanceInVault());
  }

  underlying.safeTransfer(msg.sender, underlyingAmountToWithdraw);

  // update the withdrawal amount for the holder
  withdrawals[msg.sender] = withdrawals[msg.sender].add(underlyingAmountToWithdraw);
  emit Withdraw(msg.sender, underlyingAmountToWithdraw);
}
```

Armed with our knowledge of how `underlyingBalanceWithInvestment()`, we can see that the calculation

```c
uint256 underlyingAmountToWithdraw = underlyingBalanceWithInvestment()
    .mul(numberOfShares)
    .div(totalSupply);
```

Is once again relying on the underlying value of yCRV with respect to the yAsset in the yPool

By reversing the price change created by selling a large sum of yUSDC in the pool, this maximises the `underlyingBalanceWithInvestment()`, increasing the `underlyingAmountToWithdraw`

### Demonstration

Below is a demonstration of this attack in action; I forked the Ethereum mainnet at the block before the attack, and launched my own:

![terminal](/images/terminal.png)

The left hand terminal is the local block chain outputting information using hardhat's `console.log`, and the right hand terminal is the deployer script (which also executes the deployed script) creating many transactions which drain the fUSDC vault and increase its own USDC and USDT balances.

## Aftermath

By abusing the way Harvest Finance's fUSDC vault calculated the amount of underlying asset its LP was worth, it fell victim to a flash-loan oracle manipulation attack.

The exploiter manipulated the yCRV : yUSDC pair in order to drain Harvest Finance's vaults of its uninvested USDC, and then proceeded to do the same with yCRV : yUSDT.

They made their getaway with approximately 11 million USD in stablecoin.
