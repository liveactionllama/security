# Comitium rewards hotfix - how to facet a Diamond Contract

## Summary

- **Published date** 2022-04-07
- **Funds at risk** no
- **Funds lost** no

This was our first security issue that we had to solve.

First of all:

- no funds were lost
- no user was in danger of losing their funds
- the protocol was not in danger of losing any funds

We were notified there is an issue in the rewards pipeline regarding our [Comitium](https://gov.fiatdao.com/senatus/overview) **if and only if** we update the rewards contract.

This allowed us to properly spend the time to understand the bug (which could become exploitable) plan for a fix, test the fix in a forked mainnet state, deploy and also test on mainnet.

## Background

Our rewards schedule was coming to an end, thus we planned to update the rewards contract. Our system is kind of complicated not only because there is a considerable amount of code, but mostly because the Comitium uses [EIP-2535](https://eips.ethereum.org/EIPS/eip-2535) known as the "Diamond Pattern" or "Diamonds, Multi-Facet Proxy".

I have to explain how "The Diamond" works because this is crucial to understand and a central theme througout this disclosure.

### Diamond Pattern

The **EIP-2335** is a multi-facet proxy, which means that multiple implementations can be defined for a single instance. This pattern has multiple advantages, like going over the contract limit of the EVM of 24576 bytes (0x6000 in hex) imposed by [EIP-170](https://eips.ethereum.org/EIPS/eip-170), organizing code and data in separate contracts, and in our case, what was really useful (but also contributed to creating the problem) swapping functionality in and out of the proxy.

Explained another way, the Diamond is a proxy that can point to different implementations depending on the function that is called. It first identifies the function that is called, then it points to the implementation of that function. Thus, it has a mapping of function signatures to implementations, and delegates matched calls to each implementation.

This primer should get you started with the Diamond Pattern, if you're interested in finding more, the [EIP-2535](https://eips.ethereum.org/EIPS/eip-2535) is the best place to start.

---

Our [Comitium](https://github.com/fiatdao/comitium) repository is a fork of [BarnBridge's](https://barnbridge.com/) rewards contracts [BarnBridge-Barn](https://github.com/BarnBridge/BarnBridge-Barn), since this helped us get started quickly and we had a lot more time working on [the core product](https://github.com/fiatdao/fiat) instead of the governance.

This means that both repositories and [any fork](https://github.com/BarnBridge/BarnBridge-DAO/network/members) has the same possible vulnerability.

## Details of vulnerability

We now have to dive into the details of the code, and see when and where the vulnerability happens.

This is how the system works at the moment, and how it is supposed to work.

The user's journey starts when they want to deposit FDT tokens into the system to get voting power and rewards. The deposit functionality is implemented in [ComitiumFacet.sol](https://github.com/fiatdao/comitium/blob/main/contracts/facets/ComitiumFacet.sol).

The user first calls the `deposit()` method to enter the system with a number of tokens.

Let's have a closer look at what happens when the method is called.

```solidity
// deposit allows a user to add more fdt to his staked balance
function deposit(uint256 amount) public {
    require(amount > 0, "Amount must be greater than 0");

    LibComitiumStorage.Storage storage ds = LibComitiumStorage.comitiumStorage();
    uint256 allowance = ds.fdt.allowance(msg.sender, address(this));
    require(allowance >= amount, "Token allowance too small");

    // this must be called before the user's balance is updated so the rewards contract can calculate
    // the amount owed correctly
    if (address(ds.rewards) != address(0)) {
        ds.rewards.registerUserAction(msg.sender);
    }

    uint256 newBalance = balanceOf(msg.sender).add(amount);
    _updateUserBalance(ds.userStakeHistory[msg.sender], newBalance);
    _updateLockedFdt(fdtStakedAtTs(block.timestamp).add(amount));

    address delegatedTo = userDelegatedTo(msg.sender);
    if (delegatedTo != address(0)) {
        uint256 newDelegatedPower = delegatedPower(delegatedTo).add(amount);
        _updateDelegatedPower(ds.delegatedPowerHistory[delegatedTo], newDelegatedPower);

        emit DelegatedPowerIncreased(msg.sender, delegatedTo, amount, newDelegatedPower);
    }
    ds.fdt.transferFrom(msg.sender, address(this), amount);

    emit Deposit(msg.sender, amount, newBalance);
}
```

Breaking down the method we see a few different actions that have to happen for a successful deposit.

The user needs to enter the system with a positive amount of tokens:

```solidity
require(amount > 0, "Amount must be greater than 0");
```

The storage is initialized. This is uncommon, but it is required when working with Diamond contracts since the storage layout has to be handled differently.

```solidity
LibComitiumStorage.Storage storage ds = LibComitiumStorage.comitiumStorage();
```

If you want to understand how the storage is retrieved, check the implementation of [`comitiumStorage`](https://github.com/fiatdao/comitium/blob/9bf8a9ee08e837eba3941a97df47da5c14dfdd5f/contracts/libraries/LibComitiumStorage.sol#L42-L47).

The storage layout is defined in [LibComitiumStorage.sol](https://github.com/fiatdao/comitium/blob/main/contracts/libraries/LibComitiumStorage.sol) and looks like this:

```solidity
struct Storage {
    bool initialized;

    // mapping of user address to history of Stake objects
    // every user action creates a new object in the history
    mapping(address => Stake[]) userStakeHistory;

    // array of fdt staked Checkpoint
    // deposits/withdrawals create a new object in the history (max one per block)
    Checkpoint[] fdtStakedHistory;

    // mapping of user address to history of delegated power
    // every delegate/stopDelegate call create a new checkpoint (max one per block)
    mapping(address => Checkpoint[]) delegatedPowerHistory;

    IERC20 fdt;
    IRewards rewards;
}
```

Calling `ds = LibComitiumStorage.comitiumStorage()` retrieves the pointer of the structure defined above in the `ds` variable.

Next, the allowance is checked to be at least or equal to the amount of tokens the user wants to deposit:

```solidity
uint256 allowance = ds.fdt.allowance(msg.sender, address(this));
require(allowance >= amount, "Token allowance too small");
```

Using the pointer to the data structure, we have access to the `fdt` token address. Having the token address allows us to check the approved allowance (`ds.fdt.allowance()`). This is something that it's not required to do, since the `transferFrom` method will fail later if the amount is not enough.

As I said previously, these contracts were forked, we did minimal changes to them.

Next the rewards contract is notified the user entered the system. This is the important, critical part of the system where the bug existed:

```solidity
// this must be called before the user's balance is updated so the rewards contract can calculate
// the amount owed correctly
if (address(ds.rewards) != address(0)) {
    ds.rewards.registerUserAction(msg.sender);
}
```

Even the comments say that the rewards contract has to be called for the owed amount to be calculated correctly.

This is exactly where the problem appears since we planned to update the rewards contract. This means that by replacing one rewards contract with another, and having the same code we have above, both of the rewards contracts will not be notified that the user entered the system. That means that new people who deposit will not be marked correctly in the old rewards contract.

The call stack when `Rewards.registerUserAction()` is called is a bit too complicated to explain it exhaustively in here, but the important part to understand is that when the user deposits tokens, the rewards contract saves a "checkpoint" of the global multiplier. This multiplier is then used to calculate how many rewards were accrued over time. This whole thing is also easier to understand if you know that the multiplier is always increased. Thus, the reward is the difference between the current multiplier and the previously snapshotted multiplier when the user deposited tokens.

Having the knowledge of understanding how the rewards are calculated (based on the difference between the current and previous multiplier) and the fact that when replacing the rewards contract, the old rewards contract will not update the multiplier for the user, we can now see how the bug would be triggered.

After the rewards contract is updated, a new user entering the system will not be initialized in the old rewards contract. Thus, the new user can withdraw rewards from the old rewards contract. This doesn't happen for existing users, because they already are initialized in the old rewards contract.

Thus, the funds giving out rewards were never at risk since we did not replace the rewards contract without a fix.

## Details of fix

Once we understood the problem, we started thinking about different ways to fix the issue.

We had a few options:

**Exploiting the contracts ourselves**

We could replace the rewards contract, calculate how many tokens we would need to deposit to get all rewards, deposit FDT tokens and claim immediatelly the rest of the rewards.

This means that any new user trying to use the old rewards contract to obtain their own rewards would fail, since there are no remaining rewards in the system.

To complete this fix we would need to obtain a snapshot of the current rewards any user can claim and find a way to distribute those rewards to the users. This is totally possible but sounded risky (running the exploit) and also required deploying an airdrop system, contract handling airdrop and the backend providing the Merkle proof for each user.

This seemed like a lot of risky work since we had to exploit our own contracts (also questionable), create the snapshot code, and still maintain the backend providing the Merkle proof for an unknown time period, until all users claim all tokens.

**Updating the Diamond with a fix**

Another way would be to make sure the old rewards contract is still called, even after we replace it with the new one. Of course, the new rewards contract will still be called, since each contract needs to initialize the "multiplier" for any user entering the system.

This fix only requires to update the `ComitiumFacet.sol` contract, no change to the `Rewards.sol`.

Because we are effectively doing an upgrade, we need to be mindful of the storage we are about to change (if any). Also, since a Diamond pattern is used, storage management is bit more complicated since one needs to understand the common development patterns.

Also, we are not allowed to add any storage variables in any of the facets since these will not be seen by the Diamond when the `delegatecall` happens.

If the storage talk doesn't make a lot of sense, don't worry. It's complicated for everybody.

Even though this seems more complicated, we liked this approach because it doesn't require hacking our own system, and deploying airdrop contracts. Also once the fix was done, the reward contracts can both be used the same by the users and by any other systems claiming rewards. Basically, once the fix happens, the everything works as expecter, no actor has to alter their behaviour.

---

Once we had a good understanding of the problem and a few strategies on how to fix the issue, we had a final meeting to decide what path to take. We finally picked **Updating the Diamond with a fix**.

But we had to be very careful. Thankfully, we knew how to properly test, how to update the Diamond and had access to the right people that could review our fix before it hits the mainnet.

We started by creating a repository that would fork the existing blockchain state and simulate a fix starting from that point. This way we can test the fix before it hits the mainnet. I won't go into each detailed test we did, but we made the repository public.

However, if you are interested in the code, you can find it [here](https://github.com/fiatdao/rewards-sim)

There are 3 tests that simulate different things:

- [`test_claim()`](https://github.com/fiatdao/rewards-sim/blob/82e95cdc4fcc93f2ffd8a4d02556a53d9991fa77/src/Rewards.t.sol#L108) - claims the rewards without any change (no rewards updated, no fix)
- [`test_setNewRewards()`](https://github.com/fiatdao/rewards-sim/blob/82e95cdc4fcc93f2ffd8a4d02556a53d9991fa77/src/Rewards.t.sol#L145) - claims the rewards after replacing the rewards contract (rewards updated, no fix)
- [`test_setNewRewards_withUpdatedComitium()`](https://github.com/fiatdao/rewards-sim/blob/82e95cdc4fcc93f2ffd8a4d02556a53d9991fa77/src/Rewards.t.sol#L209) - claims the rewards after replacing the rewards contract and updating the comitium contract (rewards updated, fix deployed)

Feel free to check how each test was done, but running the tests needs a more complicated setup (forking the chain, and starting from an older block) that is not detailed in the repository.

We discussed with people who were part of the comitium and rewards repo, [the creator of the Diamond Proxy](https://twitter.com/mudgen) (because he is an advisor), the rest of our team and we carefully reviewed the fix and the tests.


The fix can be inspected in this [pull request](https://github.com/fiatdao/comitium/pull/7). And it's quite simple when all of the other extra steps were done correctly. We replaced the `deposit()` method in the `ComitiumFacet.sol` contract.

The address of the old rewards contract is defined in `ComitiumFacet`:

```solidity
IRewards constant rewardsOld = IRewards(0x2458Fd408F5D2c61a4819E9d6DB43A81011E42a7);
```

The old rewards contract is defined as a `constant` because it is very important not to be in the contract's storage, since only the contract's bytecode is used (not the storage).

The `deposit()` method was updated, and this part of the code was altered:

```solidity
// this must be called before the user's balance is updated so the rewards contract can calculate
// the amount owed correctly
if (ds.firstRegisterUserAction[msg.sender] == false) {
    rewardsOld.registerUserAction(msg.sender);
    ds.firstRegisterUserAction[msg.sender] = true;
}
if (address(ds.rewards) != address(0)) {
    ds.rewards.registerUserAction(msg.sender);
}
```

You can see the old contract rewards is called once and only once for a new user. The rest is the same, only this part was added:

```solidity
if (ds.firstRegisterUserAction[msg.sender] == false) {
    rewardsOld.registerUserAction(msg.sender);
    ds.firstRegisterUserAction[msg.sender] = true;
}
```

To optimize gas usage for our users, we created the mapping `firstRegisterUserAction` in our storage library. This way we don't do external calls for the same user twice. This saves gas when the same user adds additional tokens in the system:

```solidity
mapping(address => bool) firstRegisterUserAction;
```

We finally gathered the multisig owners and deployed the fix to the mainnet with no hiccups and without creating any issue in how the users interact with the protocol, and without having any funds at risk at any moment.

## Timeline of events

- **2022-03-09**: We found out there is some kind of bug related to updating the rewards.
- **2022-03-10**: We started to gather info related to the bug.
- **2022-03-11**: We discussed options and strategies that fix our problem. Picked a solution and started working on it.
- **2022-03-15**: Since we weren't time pressured, we took time to properly create the setup, test the fix, ask for reviews and optimize as much as we could. 
- **2022-03-17**: We deployed the fix that changes the [`deposit()`](https://etherscan.io/tx/0xfe97da090a5aa8e24a3288325d79fb8f242b47e02eebb6603cf9c0062577d327/advanced) method.

## Links

- [Comitium repository](https://github.com/fiatdao/comitium)
- [Rewards old](https://etherscan.io/address/0x2458fd408f5d2c61a4819e9d6db43a81011e42a7)
- [Rewards new](https://etherscan.io/address/0x314D7Df34B72E485a5c5fDFAeFF7DEd6335B3a6e/)
- [Tx that replaces `deposit()`](https://etherscan.io/tx/0xfe97da090a5aa8e24a3288325d79fb8f242b47e02eebb6603cf9c0062577d327)
- [EIP-2535](https://eips.ethereum.org/EIPS/eip-2535)
- [Simulation of fix repository](https://github.com/fiatdao/rewards-sim)
- [Pull request with Comitium fix](https://github.com/fiatdao/comitium/pull/7/files)
- [Comitium UI](https://gov.fiatdao.com/senatus/overview)
