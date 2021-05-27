## DIP20
---
DIP20 token canisters manage and track *[fungible tokens](https://www.markdownguide.org)*. fungible tokens are completely interchangeable with each other. They hold the same value and are used for making payments and tracking balances. This is the vast majority of crypto currencies like BTC, and ETH.

### Creating an DIP20 Token Canister
---
*[Canisters](https://www.markdownguide.org)* allow us to easliy create our own DIP20 Token,


![Screenshot](img/DIP20Init.png#zoom#shadow)


Here is how it might look minting your tokens
<div class="bd-example"><pre><code>
dfx canister call token initialize "'(\"test token\", \"TT\", 0, 1000)'"
true
</pre></code></div>

We're creating an inital supply of tokens that will be deposited to the *[Caller](https://www.markdownguide.org)* of the initialize method.

That’s it! Once initialized, we will be able to query the total supply:
<div class="bd-example"><pre><code>
dfx canister call token totalSupply '()'
(1_000)
</pre></code></div>

We can also transfer these tokens to other accounts and query balances:

<div class="bd-example"><pre><code>
dfx canister call token transfer (Principle, amount)
dfx canister call token balanceOf (Principle)
</pre></code></div>

##Preset DIP20 Canister
[DIP20](https://github.com/ALLiDoizCode/DIP20)

This canister is ready to deploy without having to write any Motoko code. It can be used as-is for quick prototyping and testing, but is also suitable for production environments.