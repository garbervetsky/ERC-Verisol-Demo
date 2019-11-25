## Formal Verification of ERC20 implementations with VeriSol 

In a previous [post](https://forum.openzeppelin.com/t/openzeppelins-online-erc20-verifier-behind-the-scenes/1675) @tinchoabbate showed implementation details on [the ERC online verifier](https://erc20-verifier.openzeppelin.com/), a tool
to check compliance with the ERC20 standard. 
Essentially, it checks for a series of predefined rules that an ERC20 implementation should comply, like proper function definitions, use of events, measures to avoid front running attacks, among others.

It is really nice to have automated tools that help developers to produce complaint code and reduces the risk of vulnerabilities before deploying the code. However, as @tinchoabbate mentioned in the post:

>**the ERC20 Verifier does not check behavior**
>**the `transfer` function of the token might as well be stealing all tokens, and this tool wouldn’t even notice.**

This is also warned in @spalladino when he introduces [the ERC online verifier](https://erc20-verifier.openzeppelin.com/):


> Note that the script **does not verify that the functions found behave as expected** . It just checks for matching signatures, return types, existence of custom modifiers, event emissions, among others.

This is because the tool is not meant for checking functional properties. Checking functional properties is much harder.
On one hand they are more difficult to formalize: think about how to write a property prescribing that tokens cannot be stolen in some logic or a property allowing *only* safe cases of reentrancy. 
On the other hand they are also harder to verify due to several factors including decidability of the logic used in the specification (see [this wikipedia](https://en.wikipedia.org/wiki/Decidability_(logic)) summary), scalability (large contracts, the blockchain itself), lack of context (who and how is going to invoke the contract), etc.

Even though it may not be possible or feasible to formalize all functional and security properties (yes, we still need human audits!) there are several properties that worth verifying, as we want to thoroughly test our contracts in order to gain more confidence.
Even though testing does not guarantee the absence of bugs, a good test suite not only helps to better validate our contracts, it also serve for regression purposes.
In that sense, verification can be thought as testing a set of properties (not all, only the ones we formalize) for *all* possible scenarios (including not only parameters, also message senders, blockchain state, storage, etc).

As an example of functional specifications, some properties of interest for an ERC20 implementation can be:

1) A *function level contract* like stating that the function `transfer` decreases sender balance in a determined `amount` and increases the destination balance in the same `amount` without affecting other balances. The sender must have enough tokens to perform this operation.

2) A *contract level invariant* prescribing that the sum of balances is always equal to the total supply.

3) A *temporal property* specifying a property that must hold for a sequence of transactions to be valid:  e.g, `totalSupply` does not increase unless `mint` function is invoked.

For checking the properties we will use the tool [VeriSol](https://github.com/microsoft/verisol), a research prototype developed by Microsoft Research. 
This tool takes contracts written in  Solidity and tries to prove the contract satisfies a set of given properties or provides a sequence of transactions that violates the properties. 
VeriSol includes a Code Contracts library  written in Solidity (coined from [.NET Code Contracts](https://docs.microsoft.com/en-us/dotnet/framework/debug-trace-profile/code-contracts)) to augment the specification capabilities beyond Solidity provided ‘assert’ and ‘require’.

For the purposes of this post I am working with [this commit](https://github.com/microsoft/verisol/commit/6298e0ebc7499bdda1f076f97d81ba0e07ccf07e), so I would recommend to [build the tool](https://github.com/microsoft/verisol/blob/master/INSTALL.md#install-from-sources) using this version or from a more recent commit instead of using the precompiled version.   

I took the examples of ERC20 implementation given in [VeriSol regression suite](https://github.com/microsoft/verisol/tree/master/Test/regressions) which are slight adaptations of the original [ERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC20) implementation from [Open Zeppelin](https://openzeppelin.com/). 
All the examples can be found [here](https://github.com/garbervetsky/ERC-Verisol-Demo/).


## Working with function level contracts 

The function  [`transfer`](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-0.sol#L73) decreases sender balance in a determined `amount`  and increases the destination balance in the same `amount` without affecting other balances. The sender must have enough token to perform this operation.

The function `transfer` equipped with assertions looks like this. 

```
function transfer(address recipient, uint256 amount) public returns (bool) {
  _transfer(msg.sender, recipient, amount);
  assert (VeriSol.Old(_balances[msg.sender] + _balances[recipient]) == _balances[msg.sender] + _balances[recipient]);
  assert (msg.sender == recipient || ( _balances[msg.sender] == VeriSol.Old(_balances[msg.sender] - amount));

  return true;
}
```

The spec is straightforward. We expect the balance of the `recipient` to be increased by `amount` and that amount of tokens should be decreased form the balance of `msg.sender`. 
Note that VeriSol provides the special construct (from the contract library) `VeriSol.Old(expr)` to denote the value on an `expr` *before* the invocation. 
The first assertion states that the sum of the balances of `sender` and `recipient` before calling `transfer` is the same than after calling `transfer`.
The second assertion consider two cases: i) when the recipient and the sender is the same and nothing changes or ii) when the `amount` is decreased (and the should be increased accordingly to comply with the previous assertion).

What follows is the code snippet of the [`_transfer`](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-0.sol#L168) function that perform the actual transference of token.

```
function _transfer(address sender, address recipient, uint256 amount) internal {
  require(sender != address(0), "ERC20: transfer from the zero address");
  require(recipient != address(0), "ERC20: transfer to the zero address");

  _balances[sender] = _balances[sender].sub(amount);
  _balances[recipient] = _balances[recipient].add(amount);
  emit Transfer(sender, recipient, amount);
}
```

By executing:  `Verisol ERC20-0.sol ERC20` we get:
```
	*** Proof found! Formal Verification successful! (see boogie.txt)
```

meaning that VeriSol was able to prove the contract complains with the specification.

It is worth noticing that what we have just  proven is a *partial* specification since there is an important property missing: the balances of *all* other accounts must remain unchanged!
Indeed, an implementation of `_transfer` that changes the balance of any other accounts will be undetected by VeriSol with this specification.
The specification must explicitly state that the balance of all other accounts remains unchanged.
This property is not as simple as the previous one, since it must *quantify*  over the set of all accounts in the mapping of balances. However, the assertion language in Solidity neither provides means to quantify over set of the elements nor it is possible to iterate over all the keys in a mapping. Therefore, one cannot even express and test this property in Solidity. 

Verification-aware languages like [JML](https://www.openjml.org/) or [Dafny](https://github.com/dafny-lang/dafny) introduced clauses like `asignable` or `modifies` to define the part of the state than can be change. VeriSol has recently included preliminary support of a means specify frame conditions. They provide a new clause `VeriSol.Modifies` that looks like this:  `Verisol.Modifies(mapping,[k1, ..., kn])`. That will internally translate into a frame condition stating that elements in the mapping for the indicating keys `[k1, …, kn]` may be changed but the remaining elements must not be changed.
For instance, consider 
[this version of `transfer`](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-mod1.sol#L78), similar than the previous one, but including an incorrect frame condition `VeriSol.Modifies(_balances, [recipient]);` 

When we execute `VeriSol ERC20-mod1.sol ERC20` we get: 
``` 
	*** Did not find a proof (see boogie.txt)
	*** Found a counterexample (see corral.txt)
	-----Transaction Sequence for Defect ------
./ERC20-mod1.sol(44,1): : ERC20::Constructor (this = address!0, msg.sender = address!4, msg.value = 11, totalSupply = 655)
./ERC20-mod1.sol(73,1): : ERC20::transfer (this = address!0, msg.sender = address!4, msg.value = 12, recipient = address!3, amount = 266)
./ERC20-mod1.sol(80,1): : ASSERTION FAILS!
```

which points to the incomplete modifies clause. When we fix the modifies clause (see [this version of `transfer`](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-mod2.sol#L78) with the right frame condition `VeriSol.Modifies(_balances, [mgs.sender, recipient]);`  we obtain:

```
	*** Proof found! Formal Verification successful! (see boogie.txt)
```

To show that VeriSol can find errors in the code, let's assume we forgot to [update the balance](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-1.sol#L174) of the sender.

```
function _transfer(address sender, address recipient, uint256 amount) internal {
  require(sender != address(0), "ERC20: transfer from the zero address");
  require(recipient != address(0), "ERC20: transfer to the zero address");

  // _balances[sender] = _balances[sender].sub(amount);
  _balances[recipient] = _balances[recipient].add(amount);
  emit Transfer(sender, recipient, amount);
}
```

By executing `Verisol ERC20-1.sol ERC20` we get:
```
	*** Did not find a proof (see boogie.txt)
	*** Found a counterexample (see corral.txt)
	-----Transaction Sequence for Defect ------
./ERC20-1.sol(44,1): : ERC20::Constructor (this = address!0, msg.sender = address!6, msg.value = 12, totalSupply = 1)
./ERC20-1.sol(73,1): : ERC20::transfer (this = address!0, msg.sender = address!3, msg.value = 13, recipient = address!4, amount = 142)
./ERC20-1.sol(75,1): : ASSERTION FAILS!
```

which means there is one sequence of transactions (i.e., create, transfer) which makes the
 `assert (VeriSol.Old(_balances[msg.sender] + _balances[recipient]) == _balances[msg.sender] + _balances[recipient]);` on line 75 fail.

## Checking overflows and underflows

As a bonus track, let us suppose a much simpler specification stating that after transferring tokens the `sender` [must have less or equal amount of tokens](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-uf.sol#L76) and resp. the `recipient` [must have more or equal amount of tokens](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-uf.sol#L77).

```
function transfer(address recipient, uint256 amount) public returns (bool) {
  _transfer(msg.sender, recipient, amount);
  assert (_balances[msg.sender] <= VeriSol.Old(_balances[msg.sender]));
  assert (_balances[recipient] >= VeriSol.Old(_balances[recipient]
  return true;
}
```

Even though this spec is very weak, it can help to uncover an overflow/underflow bug, for instance by [forgetting to use `SafeMath`](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-uf.sol#L173).

```
function _transfer(address sender, address recipient, uint256 amount) internal {
  require(sender != address(0), "ERC20: transfer from the zero address");
  require(recipient != address(0), "ERC20: transfer to the zero address");

   _balances[sender] = _balances[sender] - amount; // bug not using safemath
  //_balances[sender] = _balances[sender].sub(amount);
  _balances[recipient] = _balances[recipient].add(amount);
  emit Transfer(sender, recipient, amount);
}
```

We need to include the flag  `/useModularArithmetic` to tell VeriSol to use modular arithmetic instead of unbounded integers. This flag is not used as default because it comes with a performance overhead, but required for soundness. 

`Verisol ERC20-uf.sol ERC20 /useModularArithmetic`

```
	*** Did not find a proof (see boogie.txt)
	*** Found a counterexample (see corral.txt)
	-----Transaction Sequence for Defect ------
./ERC20-demo-under.sol(43,1): : ERC20::Constructor (this = address!0, msg.sender = address!10, msg.value = 11, totalSupply = 115792089237316195423570985008687907853269984665640564039457584007913129639935)
./ERC20-demo-under.sol(75,1): : ERC20::transfer (this = address!0, msg.sender = address!3, msg.value = 13, recipient = address!4, amount = 1)
./ERC20-demo-under.sol(77,1): : ASSERTION FAILS!
```

which is exactly the assertion we included. 

## Working with contract invariants 

 Consider a *contract level invariant* prescribing that the sum of balances is equal to the total supply. In contrast to the previous property which applied only to the `transfer` function, now we want to describe a property that must hold thorough *all* the transactions of the contracts. Thus, this property must be checked after finishing each transaction.

Fortunately, VeriSol includes a way to specify contract invarians using a special function [`contractInvariant`](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-2.sol#L271). Here is an example:

```
function contractInvariant() private view {
  VeriSol.ContractInvariant(_totalSupply == VeriSol.SumMapping(_balances));
}
```

Another cool aspect of VeriSol is the built-in clause `SumMapping` which allows expressing the sum of all values in a mappings, which is handy for our specification purpose (recall the previous discussion about quantifying over all the elements in a mapping).

Suppose, we inadvertently forgot to update the `totalSupply` in the [`_mint`](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-2.sol#L215) function:

```
function _mint(address account, uint256 amount) internal {
  require(account != address(0), "ERC20: mint to the zero address");

  // _totalSupply = _totalSupply.add(amount);
  _balances[account] = _balances[account].add(amount);
  emit Transfer(address(0), account, amount);
}
```

Executing `Verisol ERC20-2.sol ERC20` we get:
```
  *** Did not find a proof (see boogie.txt)
  *** Found a counterexample (see corral.txt)
	-----Transaction Sequence for Defect ------
/ERC20-demo2.sol(43,1): : ERC20::Constructor (this = address!0, msg.sender = address!6, msg.value = 14, totalSupply = 1)
/ERC20-demo2.sol(90,1): : ERC20::mint (this = address!0, msg.sender = address!3, msg.value = 15, account = address!4, amount = 583)
/ERC20-demo2.sol(46,1): : ASSERTION FAILS!
	---------------
  ```

VeriSol reports finds the problem and reports the right transaction sequence (the problem is due to a bug in `mint`). There is a minor issue in that instead of reporting the line corresponding to the contract invariant it wrongly reports the bug in the last line of the constructor (this [issue](https://github.com/microsoft/verisol/issues/166) has been already reported).

I strongly recommend developers to reason about and specify contracts invariants, because they really help to understand the relationship between the contract storage variables and help  maintain the contract in a consistent state. Note that general issues like overflows/underflow can be also be discovered using invariants, since an overflow/underflow may break an invariant.

For instance, the same problem of underflow in the [`_transfer`](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-uf2.sol#L169) function can be uncover by the failing [invariant](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-uf2.sol#L242). 

Recall that in order to check overflows we need to use the additional flag `Verisol ERC20-uf2.sol ERC20 /useModularArithmetic`:

```
	*** Did not find a proof (see boogie.txt)
  *** Found a counterexample (see corral.txt)
	-----Transaction Sequence for Defect ------
./ERC20-demo.sol(43,1): : ERC20::Constructor (this = address!0, msg.sender = address!6, msg.value = 13, totalSupply = 2)
./ERC20-demo.sol(75,1): : ERC20::transfer (this = address!0, msg.sender = address!4, msg.value = 14, recipient = address!3, amount = 115792089237316195423570985008687907853269984665640564039457584007913129635749)
./ERC20-demo.sol(46,1): : ASSERTION FAILS!
```

The file [ERC20.sol](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20.sol) contains the full specification for the function `transfer` as well as the contract invariant.  

Other tools like [VerX](https://verx.ch/) can also verify to (multi) contract invariants. It also includes support for specifying and checking temporal properties (see bellow). 
Unfortunately, unlike VeriSol the tool is not open and requires a payed subscription for its use.

## Working with temporal properties 

A *temporal property* is a property that must hold over a sequence of transactions:  e.g, `totalSupply` does not increase unless `mint` function is invoked.

One can specify such properties in [temporal logic LTL](https://en.wikipedia.org/wiki/Linear_temporal_logic).
(Very) roughly speaking, a LTL logic extends a classical logic with relations about states, so with LTL we can express properties about sequences of states (i.e., a sequence of transactions). For instance, one can say specify that the  value of `_totalSupply` does not change unless you apply a `mint` operation, or once the `_totalSupply` is greater than zero, it cannot decrease to zero, or `mint` cannot be applied twice, etc.  
To specify temporal properties, we will resort on [VeriMan](https://github.com/VeraBE/VeriMan) a tool developed by @veraBE. It takes a Solidity contract and a set of formulas written in PTLTL (Past Linear Temporal Logic, more details [here](https://forum.openzeppelin.com/t/veriman-a-prototype/1446)). 
The tool instruments the contract to find a trace that falsifies at least one of the properties or prove that they hold. The instrumented contract can be checked with tools like VeriSol, Mythril, Manticore, Echidna, etc. In the case of VeriSol, we can include in the formulas expressions like `Verisol.Old(_totalSupply)` which can be handy for some properties.  

For instance, the formula `(VeriSol.Old(_totalSupply) ==_totalSupply || mintCalled)` states that for all transaction sequences,  `_totalSupply` remains the same unless the `mint` function is called. 

When we invoke VeriMan we obtain the following counter example. 
```
[.] Running VeriSol
[!] Counterexample found: ['Constructor']
```

which looks counter intuitive at the beginning. However, it is indeed true that the constructor changes the value of `_totalSupply` and 
`VeriSol.Old` is implemented in a way that its value is undefined before creating the contract (which makes sense!).
So, we slightly modify the contract to add a Boolean variable that helps us avoid this particular case (the code is [here](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/ERC20-Veriman.sol) and using this [config](https://github.com/garbervetsky/ERC-Verisol-Demo/blob/master/config.json) file for VeriMan). 

Now, we invoke VeriMan with this property `notConstructor -> (VeriSol.Old(_totalSupply) ==_totalSupply)` and obtain a new counter example. 
```
[.] Running VeriSol
[!] Counterexample found: ['Constructor', 'burn']
```
which makes sense as the `burn` operation modifies the `_totalSuply'.


Now, we invoke VeriMan with this property `notConstructor -> (VeriSol.Old(_totalSupply) ==_totalSupply || mintCalled || burnCalled)` which is verified. 
```
[.] Running VeriSol
[!] Contract proven, asserts cannot fail
```


## Conclusions

We have shown how we can use a formal verification tool like VeriSol to check functional properties for a ERC20 implementation. We discussed a range of properties like function pre/postconditions, contract invariants and temporal properties.
Even though the use of these tools cannot replace professional security audit, it can help developers (and auditors) to reason about their contract and gain confidence. 

## Acknowledgments 

Special thanks to Shuvendu Lahiri for the technical assistance on VeriSol and @VeraBe for helping me with VeriMan.
