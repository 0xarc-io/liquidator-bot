# ARC Liquidator Bot

## Background

ARC is a synthetic asset protocol that takes any form of collateral in a pool and spits out debt based on the amount of collateral provided. A simple example:

1. LINK is worth $10
2. Chad deposits 10 LINK worth $100 inside ARC
3. The current c-ratio is 200%, however to be safe Chad goes with a 500% c-ratio and mints 20 LINKUSD
4. Chad now has a new position inside the LINKUSD pool with 10 LINK as collateral and 20 LINKUSD as debt.

ARC uses ChainLink oracles to determine the price of the asset.

## Technical Architecture

Each ARC Synthetic Asset has it's own set of contracts. Outlined below are the contracts and the role they play:

- ArcProxy: this is a generic proxy which has an admin and implementation. The implementation of this contract is Core. No user funds are held here, only fees earned by ARC.
- CoreV1: the implementation set for the proxy. This can be upgraded and the logic changed to whatever is required. No state is held inside this contract.
- StateV1: holds state about all the positions currently inside the pool and has logic around calculating ratios etc.
- SyntheticToken: the actual synth itself. All the collateral (LINK) is held inside here.

## Problem

The core issue that this program needs to solve is the fact that when prices drop, someone needs to liquidate the user and pay off their debt using the collateral held in their position.

Here's a test case of how this works:

```
it('should be able to be liquidate when the price decreases', async () => {
    // The price of LINK is $20, the user has $200 of collateral and
    // currently has $1000 of debt giving a 500% c-ratio.

    // The price of LINK now flash crashes to $5 throwing the user underwater
    await updateOraclePrice(ArcDecimal.new(5));

    // The user most defenitely should not be able to borrow more money if they're underwater
    await expectRevert(
      arc._borrowSynthetic(ArcNumber.new(100), ArcNumber.new(100), userWallet, currentPosition),
    );

    // This means that...
    // - Users's 50 LINK is now worth $250
    // - User has borrowed $200 worth of debt
    // - User's c-ratio is 125%, 75% below the 200% mark
    // - We'll need to liquidate them to set the c-ratio higher by selling
    //   their collateral at a discount 10% (5% ARC + 5% Bot)

    // Oh noes, time to call up Arthur.
    await arc._borrowSynthetic(ArcNumber.new(500), ArcNumber.new(250), liquidatorWallet);
    const result = await arc.liquidatePosition(currentPosition, liquidatorWallet);

    // A run down of how we calculate how much to liquidate
    // - Debt = $200, Collateral = 50LINK = $250
    // - Maximum user can borrow = $125 @ 200% c-ratio ($250 / 2).
    // - The price of LINK = $5, however because of the 10% penalty
    //   we're going to sell LINK to the liquidator for $4.50
    // - Collateral needed to borrow $200 = ($200 * 2) / $4.5 = 88.88
    // - Delta in collateral needed = 88.88 - 50 = 38.88
    // - Since we want the user to be safe we add an extra 10% margin of safety
    // - 38.88888 * 1.1 = 42.777 (rounding errors create slight differences)
    // - The user should be left with 50 - 42.7777 = 7.222 LINK
    // - Since they got rekt we reduce their debt to $7.5
    // - The new c-ratio is now 7.222 * 5 / 7.5 = 481.4%
    // - This may sound high but this is in the worse case where a
    //   flash crash happens. The c-ratio approaches infinity as the price drops more.
    // - Safety over profits.

    // Ensure the emitted parameters are correct
    expect(result.operation).toEqual(Operation.Liquidate);
    expect(result.updatedPosition.borrowedAmount.value.toString()).toEqual('7500000000000000008');
    expect(result.updatedPosition.borrowedAmount.sign).toBeFalsy();
    expect(result.updatedPosition.collateralAmount.value.toString()).toEqual('7222222222222222224');
    expect(result.updatedPosition.collateralAmount.sign).toBeTruthy();
    expect(result.params.id).toEqual(currentPosition);

    // ARC should still have 257.222 LINK left in the system (200 from Arthur + 7.22 from rekt dude)
    expect(
      await (await arc.collateralAsset.balanceOf(arc.syntheticAsset.address)).toString(),
    ).toEqual('257222222222222222224');

    // The user can still have $200 worth of LINKUSD since they got rekt
    expect(await arc.syntheticAsset.balanceOf(userWallet.address)).toEqual(ArcNumber.new(200));

    // The liquidator will now have less LINKUSD since they had to pay the rekt dude's debt
    expect((await arc.syntheticAsset.balanceOf(liquidatorWallet.address)).toString()).toEqual(
      '307500000000000000008',
    );

    // The total LINKUSD supply be the amount the user holds + the amount arthur holds
    expect((await arc.syntheticAsset.totalSupply()).toString()).toEqual('507500000000000000008');
  });
```

## Requirement

The core goal here is to develop a liquidator that:

1. Automates the above process by using The Graph data (https://thegraph.com/explorer/subgraph/arcxgame/linkusd)
2. Relies on flash loans so the user running the bot doesn't have to hold the amount of capital needed
3. Is written in Typescript and uses the bindings provided (import from the path `@arcxgame/contracts/dist/src/types`)
4. Can adjust to use the correct gas prices based on current mempool conditions (not afraid to use 2,000 gWei if that's whats needed to get the tx through)
5. Has support for multiple ARC Synths and collateral type (can be on the look out for many pools)
6. Can handle itself correctly in the case of any downtimes, nonce failures or node connection issues.

## Addresses

`LINKUSD-ArcProxy:	      https://etherscan.io/address/0x5568Df89AF6Bc835B876fDc3B2d44ef63530e419`
`LINKUSD-Oracle:	        https://etherscan.io/address/0xeffd9768a337950cdc883682dc08583d4a3fe596`
`LINKUSD-CoreV1:	        https://etherscan.io/address/0x08db418a5234337d76c78ccf630658d4e328fc69`
`LINKUSD-SyntheticToken:	https://etherscan.io/address/0x0E2EC54fC0B509F445631Bf4b91AB8168230C752`
`LINKUSD-StateV1:	        https://etherscan.io/address/0x94F40fD4018586AaFb8A6AD95441e0b58cc4c058`