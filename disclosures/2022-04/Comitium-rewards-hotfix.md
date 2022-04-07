# Comitium rewards hotfix - how to facet a Diamond Contract

## Summary

- **Published date** 2022-04-07
- **Funds at risk** no
- **Funds lost** no

This was the first security issue that we had to solve.

First of all:

- users lost no funds
- no user was in danger of losing their funds
- the protocol was not in danger of losing any funds

We were notified there is an issue in the rewards pipeline regarding our [Comitium](https://gov.fiatdao.com/senatus/overview) **if and only if** we update the rewards contract.

Because we did not yet deploy a new rewards contract, this allowed us to properly spend the time to understand the bug (which could become exploitable), plan for a fix, test the fix in a forked mainnet state, deploy and also test on mainnet.

## Background

Our rewards schedule was coming to an end. Thus we planned to update the rewards contract. Our system is kind of complicated not only because there is a considerable amount of code but mostly because the Comitium uses [EIP-2535](https://eips.ethereum.org/EIPS/eip-2535), known as the "Diamond Pattern", "Diamond Proxy" or "Diamonds, Multi-Facet Proxy".

I have to explain how "The Diamond" works because this is crucial to understand and a central theme throughout this disclosure.

### Diamond Pattern

The **EIP-2335** is a multi-facet proxy, meaning multiple contract implementations can be defined for a single instance. This pattern has various advantages, like going over the contract limit of the EVM of 24576 bytes (0x6000 in hex) imposed by [EIP-170](https://eips.ethereum.org/EIPS/eip-170), organizing code and data in separate contracts, and in our case, what was really useful (but also contributed to creating the problem) swapping functionality in and out of the proxy.

Explained another way, the Diamond is a proxy that can point to different implementations depending on the called function. It first identifies the function that is called, and then it calls its implementation. Thus, it has a mapping of function signatures to implementations and delegates matched calls to each implementation.

This primer should get you started with the Diamond Pattern; if you're interested in finding more, the [EIP-2535](https://eips.ethereum.org/EIPS/eip-2535) is the best place to start.

---

Our [Comitium](https://github.com/fiatdao/comitium) repository is a fork of [BarnBridge's](https://barnbridge.com/) rewards contracts [BarnBridge-Barn](https://github.com/BarnBridge/BarnBridge-Barn), since this helped us get started quickly and we had a lot more time working on [the core product](https://github.com/fiatdao/fiat) instead of the governance.

This means that both repositories and [any fork](https://github.com/BarnBridge/BarnBridge-DAO/network/members) has the same possible vulnerability when the rewards contract is updated.

## Details of the vulnerability

We now have to dive into the details of the code and see when and where the vulnerability happens.

This is how the system works and how it is supposed to work.

The user's journey starts when they want to deposit FDT tokens into the system to get voting power and rewards. The deposit functionality is implemented in [ComitiumFacet.sol](https://github.com/fiatdao/comitium/blob/main/contracts/facets/ComitiumFacet.sol).

The user first calls the `deposit()` method to enter the system with any number of tokens.

Let's look at what happens when the method is called.

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

Breaking down the method, we see a few different actions that have to happen for a successful deposit.

The user needs to enter the system with a positive amount of tokens:

```solidity
require(amount > 0, "Amount must be greater than 0");
```

The storage is initialized uncommonly, but it is required when working with Diamond contracts since the storage layout has to be handled differently.

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

Next, the allowance is checked to be at least or equal to the number of tokens the user wants to deposit:

```solidity
uint256 allowance = ds.fdt.allowance(msg.sender, address(this));
require(allowance >= amount, "Token allowance too small");
```

We have access to the `fdt` token address using the pointer to the data structure. The token address allows us to check the approved allowance (`ds.fdt.allowance()`). Checking is something that it's not required to do since the `transferFrom` method will fail later if the amount is not enough.

As I said previously, we forked these contracts and made minimal changes to them.

Next, the rewards contract is notified the user enters the system. This is the critical part of the system where the bug existed:

```solidity
// this must be called before the user's balance is updated so the rewards contract can calculate
// the amount owed correctly
if (address(ds.rewards) != address(0)) {
    ds.rewards.registerUserAction(msg.sender);
}
```

The comments state that the rewards contract has to be called for the owed amount to be calculated correctly.

This is exactly where the problem appears since we planned to update the rewards contract. By replacing one rewards contract with another and having the same code we have above, Comitium will not notify both of the rewards contracts that the user entered the system. That means new people who deposit will not be marked correctly in the old rewards contract.

The call stack when `Rewards.registerUserAction()` is called is a bit too complicated to explain exhaustively here. Still, the important part of understanding is that when the user deposits tokens, the rewards contract saves a "checkpoint" of the global multiplier. This multiplier is then used to calculate how many rewards were accrued over time. This whole thing is easier to understand if you know that the multiplier is always increased. Thus, the reward is the difference between the current multiplier and the previously snapshotted multiplier when users deposit tokens.

Knowing how the rewards are calculated (based on the difference between the current and previous multiplier) and the fact that when replacing the rewards contract, the old rewards contract will not update the multiplier for the user, we can now see how the bug would be triggered.

After the rewards contract is updated, a new user entering the system will not be initialized in the old rewards contract. Thus, the new user can withdraw rewards from the old rewards contract without participating in the staking process. This doesn't happen for existing users because they are already initialized in the old rewards contract.

Thus, the funds giving out rewards were never at risk since we did not replace the rewards contract without a fix.

## Details of fix

Once we understood the problem, we started thinking about different ways to fix the issue.

We had a few options:

**Exploiting the contracts ourselves**

We could replace the rewards contract, calculate how many tokens we would need to deposit to get all rewards, deposit FDT tokens and claim the rewards immediately.

This means that any new user trying to use the old rewards contract to obtain their rewards would fail since there are no remaining rewards.

To complete this fix, we would need to obtain a snapshot of the current rewards any user can claim and find a way to distribute those rewards to the users. This is possible but sounded risky (running the exploit) and required deploying an airdrop system, contract handling airdrop and the backend providing the Merkle proof for each user.

It seemed like a lot of risky work since we had to exploit our contracts (also questionable), create the snapshot code, and maintain the backend providing the Merkle proof for an unknown time until all users claim all tokens.

**Updating the Diamond with a fix**

Another way would be to make sure the old rewards contract is still called, even after replacing it with the new one. Of course, the new rewards contract will still be called since each contract needs to initialize the "multiplier" for any user entering the system.

This fix only requires updating the `ComitiumFacet.sol` contract, no change to the `Rewards.sol`.

Because we are effectively doing an upgrade, we need to be mindful of the storage we are about to change (if any). Also, since a Diamond pattern is used, storage management is a bit more complicated since one needs to understand the common development patterns.

Also, we cannot add any storage variables in any of the facets since this storage is unavailable in the Diamond when the `delegatecall` happens.

Even though this seems more complicated, we liked this approach because it doesn't require hacking our own system and deploying airdrop contracts. Also, once the fix is done, the reward contracts can be used the same by the users and by any other system claiming rewards. Everything works as expected once the fix happens; no actor has to alter their behavior.

---

Once we had a good understanding of the problem and a few strategies to fix the issue, we had a final meeting to decide what path to take. We finally picked **Updating the Diamond with a fix**.

But we had to be very careful. Thankfully, we knew how to properly test, update the Diamond, and have access to the right people who could review our fix before it hits the mainnet.

We started by creating a repository that would fork the existing blockchain state and simulate a fix starting from a fixed point in time. This way, we can test the fix before it hits the mainnet. I won't go into each detailed test, but we made the repository public.

If you are interested in the code that tested the fix, you can find it [here](https://github.com/fiatdao/rewards-sim)

These three tests simulate different things:

- [`test_claim()`](https://github.com/fiatdao/rewards-sim/blob/82e95cdc4fcc93f2ffd8a4d02556a53d9991fa77/src/Rewards.t.sol#L108) - claims the rewards without any change (no rewards updated, no fix)
- [`test_setNewRewards()`](https://github.com/fiatdao/rewards-sim/blob/82e95cdc4fcc93f2ffd8a4d02556a53d9991fa77/src/Rewards.t.sol#L145) - claims the rewards after replacing the rewards contract (rewards updated, no fix)
- [`test_setNewRewards_withUpdatedComitium()`](https://github.com/fiatdao/rewards-sim/blob/82e95cdc4fcc93f2ffd8a4d02556a53d9991fa77/src/Rewards.t.sol#L209) - claims the rewards after replacing the rewards contract and updating the comitium contract (rewards updated, fix deployed)

Feel free to check how we developed each test, but running the tests needs a more complicated setup (forking the chain and starting from an older block) that is not detailed in the repository.

We discussed with people who were part of the Comitium and rewards repo, [the creator of the Diamond Proxy](https://twitter.com/mudgen) (because he is an advisor), and the rest of our team and we carefully reviewed the fix and the tests.


The fix can be inspected in this [pull request](https://github.com/fiatdao/comitium/pull/7). And it's quite simple when all of the other extra steps were done correctly. We replaced the `deposit()` method in the `ComitiumFacet.sol` contract.

The address of the old rewards contract is defined in `ComitiumFacet`:

```solidity
IRewards constant rewardsOld = IRewards(0x2458Fd408F5D2c61a4819E9d6DB43A81011E42a7);
```

The old rewards contract is defined as a `constant` because it is very important not to be in the contract's storage since only the contract's bytecode is used (not the storage).

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

The old contract rewards are called once and only once for a new user. The rest is the same, only this part was added:

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

We finally gathered the multisig owners and deployed the fix to the mainnet with no hiccups, without creating an issue in how the users interact with the protocol and without having any funds at risk.

## Timeline of events

- **2022-03-09**: We found out there is some kind of bug related to updating the rewards.
- **2022-03-10**: We started to gather info related to the bug.
- **2022-03-11**: We discussed options and strategies that fix our problem. We picked a solution and started working on it.
- **2022-03-15**: Since we weren't time-pressured, we took the time to create the setup properly, test the fix, ask for reviews and optimize as much as possible. 
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
