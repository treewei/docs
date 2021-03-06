# CRP Tutorial

Similar to Core Pools, Configurable Rights Pools are created from a public factory. Refer to [Addresses](../smart-contracts/addresses.md) to find the contract on your network of choice.

To deploy a new Configurable Rights Pool, call newCRP on the CRPFactory. This function takes three parameters \(cheating a bit, since two of them are big structs\):

```text
function newCrp(
    address factoryAddress,
    ConfigurableRightsPool.PoolParams calldata poolParams,
    RightsManager.Rights calldata rights
)
    external
    returns (ConfigurableRightsPool)
```

The factoryAddress is that of the BFactory \(see [Core Concepts](../protocol/concepts.md)\). The Configurable Rights Pool is the Smart Pool "wrapper" around the underlying Core Pool \(BPool\). You \(caller of newCRP\) are the controller of the Smart Pool. The Smart Pool is the controller of the Core Pool. So the Smart Pool needs to deploy a Core Pool - and requires the BFactory to do that.

This is also a bit of future-proofing. At the moment all the Core Pools are "Bronze," but at some point we will release "Silver" and "Gold" versions of the Core Pools, and corresponding factories to create them.

The next argument is PoolParams - this is where you define the structure and basic parameters of the pool, such as the tokens it will hold, their initial weights and balances, and the swap fee.

```text
struct PoolParams {
    // Balancer Pool Token (representing shares of the pool)
    string poolTokenSymbol;
    string poolTokenName;
    // Tokens inside the Pool
    address[] constituentTokens;
    uint[] tokenBalances;
    uint[] tokenWeights;
    uint swapFee;
}
```

Since the [Balancer Pool Tokens](../protocol/btoken.md) are themselves ERC20 tokens, they have symbols and names. You can set both when creating your pool.

The tokens must be addresses of conforming ERC20 tokens. Balances and weights are expressed in Wei - and the weights are denormalized, not percentages. Valid denormalized weights range from 1 to 49, since the maximum total denormalized weight is 50. \(This corresponds to a percentage range from 2% to 98%.\)

The swap fee is also expressed in Wei, as a percentage. For instance, toWei\("0.01"\) means a 1% fee.

Finally, the Rights struct defines the permissions.

```text
struct Rights {
    bool canPauseSwapping;
    bool canChangeSwapFee;
    bool canChangeWeights;
    bool canAddRemoveTokens;
    bool canWhitelistLPs;
    bool canChangeCap;
}
```

 At this point \(after calling newCRP\), we have a deployed Configurable Rights Object with all its permissions and parameters defined. But we can't do much with it - mainly because there is no Core Pool yet. We need to deploy a new Core Pool, with our Smart Pool as the controller, by calling createPool\(initialSupply\). \(There is also an overloaded version of createPool; more on that later.\) 

We've already defined the tokens and balances we want the pool to hold. When we call createPool with a value for initialSupply, it will mint _initialSupply_ Balancer Pool Tokens \(BPTs\) and transfer them to the caller, simultaneously pulling the correct amount of collateral tokens into the contract. \(They end up in the Core Pool, passed through the CRP.\)

To accomplish this, we need to allow the CRP to spend our collateral tokens, before calling createPool. For an example three-token pool, we might write:

```text
const MAX = web3.utils.toTwosComplement(-1);

// crpPool was returned from CRPFactory.newCRP()
    
await weth.approve(crpPool.address, MAX);
await dai.approve(crpPool.address, MAX);
await xyz.approve(crpPool.address, MAX);

// consume the collateral; mint and xfer 100 BPTs to caller
await crpPool.createPool(toWei('100'));
```

Now we're in business! The pool will already be set up for public swapping, and depending on the permissions settings, possibly public liquidity provision as well. The Core Pool will show up on the Exchange GUI.

Note that there is also an overloaded version of createPool, where you can specify additional parameters related to updateWeightsGradually.

```text
function createPool(
    uint initialSupply,
    uint minimumWeightChangeBlockPeriod,
    uint addTokenTimeLockInBlocks
)
```

If the pool you're creating doesn't have permission to change weights, or you accept the default values of the time parameters, you can just use the single-argument version.

_initialSupply_ can be set to a value of your choice - within min/max limits \(currently 100 - 1 billion, in ether units\).

_minimumWeightChangeBlockPeriod_ enforces a minimum time between the start and end blocks of a gradual update. _addTokenTimeLockInBlocks_ has two functions. It is the minimum time delay between weight updates \(e.g., you can specify that the weights cannot change more frequently than every tenth block, or hundredth block, etc.\) It is also used when adding a token \(if you have the AddRemoveTokens permission\), as the minimum wait time between committing a new token, and applying it \(i.e., actually adding it to the pool\).

These parameters have default values, and can only be changed by overriding them in this version of createPool. The addTokenTimeLock defaults to 500 blocks \(~ 2 hours\), and the blockPeriod defaults to 90,000 \(~ 2 weeks\). If you want to use different minimum values, be sure to set them in createPool, since they are immutable thereafter!

Also note that these only apply to gradual updates. The controller can call updateWeight directly at any time, as long as no gradual update is in progress.

