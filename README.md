## Reflections on tubby cats

In this post I'll go over the tubby cats launch and provide a data-based analysis into the mistakes that were made. The goal is to share our experiences with new projects and try to prevent them from repeating our mistakes.

This article has an extreme negative bias, since I'll mostly talk about the things that went wrong instead of the ones that went right (which were many), so please take that into account. I'll only go over the issues pertaining to the tech side, as that was what I dealt with.

## The first sin
The main mistake was to make the contract completely immutable. When writing the contract I made it as trustless as possible, removing as much trust as possible into the team.

The reason for it is that I believe it's important for every participant to know the rules of the game and know that they won't be changed from under them (everyone should have complete information to make decisions, if processes are changed in the middle that creates groups of people with privileged information), while trust minimization is important because that's the whole raison d'etre for smart contracts (and it makes it impossible to hack even if the team is compromised). However, this was a huge mistake, since it greatly reduced our flexbility and the actions we could take to correct things that happened.

As we'll find out later, all issues are just secondary to this one, since they could all have been solved had the contract been mutable by the team. Furthermore, a large amount of people bought into the NFT because of the team, so they were already placing their trust on it, thus not requiring extra trust minimization on contract level.

### Context
The tubby cats sale was an NFT sale where each tubby was sold for 0.1eth and the sale was structured into two periods:
- During the first 48 hours, people that belonged to a whitelist of 19.5k addresses could mint 1 tubby cat. The whitelist was encoded as a merkle drop.
- After the first 48 hours, anyone was able to mint at a price of 0.1eth

### First problem: inconsistent casing on addresses
When minting went live we encountered an issue where many users reported they couldn't mint. The problem was that we had stored our merkle proofs along with the whitelisted addresses as submitted, and these had inconsistent casing (some were all lowercase, some uppercase and some had mixed case) and when trying to match them with the address as reported by metamask our code ran into issues. This was pretty easy to fix, we just needed to normalize the addresses. A fix was shipped quickly (<30 mins) and everything proceeded as normal.

I wouldn't consider this a problem, it was just a minor bug we had.

### Incorrect addresses
The list of addresses we used for the whitelist had 109 addresses (out of 19.5k, so ~0.5%) that were submitted incorrectly with a space at the end. This led to the generation of an incorrect merkle tree and all those addresses were unable to mint. Due to the immutability of the contract we were unable to fix this, which had the following downsides:
- Had to spend a lot of time dealing with people that couldn’t mint during launch, when we were short on time
- Bad PR
- Had to come up with an alternate solution

The reason why this happened was that:
- A short visual check on the addresses didn’t reveal any problem (didn’t see spaces)
- We only  tested minting from subset of addresses (locally and on testnet), instead of for every single one, in order to save time
- We didn’t have the ability to make changes to merkle tree (or mint new tubbies) after deployment, so we couldn’t fix it
- I didn’t run a correctness check on the addresses, as I was interpreting them as hex and I assumed that any inconsitency (eg: casing) would be normalized through that (however I prefixed them with zeroes to make them be 32 bytes long, and the space messed up the conversion)

## Gas war
By far the biggest issues with the launch came from the gas war at the end. It also caused some misinformation to appear:
- All available tubbies in public sale were minted by bots. Incorrect! 2/3rd were minted by bots (16% of total supply)
- Bots sold at 8x. Incorrect! Biggest bot bought at 0.3ETH per tubby and sold at 0.4-0.6 ETH, so ~1.5x.
- A single entity now owns 950 tubby cats.  Incorrect, big minter seems to be a group of users that pooled funds for a bot, the tubbies were distributed afterwards among them and most have already been sold
- Botting caused the supply to be very centralized. Wrong, botting only affected 16% of total supply and gini coefficient of full distribution is actually lowish
- Contract minting allowed bot to mint 950. Wrong! Contract minting is completely unrelated to that.
- Bad code caused failed txs to be 5x more expensive. Partially correct, failed txs were more expensive by ~2x.

Let’s break them down:

### Contract minting
Minting function had no checks against minting from contract, which meant that anyone could mint from another contract, which in turn could be used to make a contract that minted tubby cats multiple times.

I’ve seen a lot of people saying that these is what allowed bots to accumulate large stacks of tubby cats, however that’s just plain wrong. If we added a check against contract minting (eg: require(tx.origin == msg.sender)) what would have happened is that bots would have sent multiple txs instead of a single one. Another solution would have been to limit the amount of tubbies that can be minted from each wallet, but then bots would have just sent txs from multiple wallets. None of these simple solutions work, since they are trivial to bypass.

Contract minting enabling bots is incorrect, however that is not to say contract minting has no issue. The real issue, which I haven’t seen anyone mention, is that minting from contract allows the miner to amortize fixed transaction costs over many mints. To understand this you need to think that every time you send a tx you pay a fixed cost that is spent on things like signature verification and then a variable cost that changes depending on the code executed.

If you are a user and send 5 txs to mint 5*5=25 tubbies you need to pay fixed tx costs 5 times and execution costs 5 times too, however if you use a contract you’ll send a single tx, thus you’ll pay execution costs 5 times too, but you only pay fixed tx costs once. This amortizes the cost over many mints, and the more NFTs you mint in a single tx the bigger the edge you get over someone minting directly.

I’ve measured how big this edge is and, in the case where you mint a very high amount of tubbies you can achieve up to a 36% reduction in gas costs. However in the specific case of tubby cats mint and the 950 botter this didn’t actually matter since they overbid their gas price by >2x compared to the lowest mint in gas price. Because the difference in gas price is much higher than the gas reduction from contract minting, there was no user that was pushed out because of it.

However for future mints it would be better to avoid it.

### Supply check
The public minting function had a check against supply at the end of the function. The consequence of this is that if someone sends a tx after all tubbies have been minted, most of the code would execute before reaching the supply check and reverting. If instead that check was at the start, execution would stop early and less gas would be consumed.

The reason why I put it at the end was to optimize for the case when minting is successful, since in that case this saves one addition operation. However, in case that check fails, gas cost becomes much higher, so this was definitely the wrong decision. 

According to my benchmarks, if the check was at the start, the reduction in gas cost for failed txs would be roughly 2x. I’ve seen people mention numbers around 5x in gas savings, I think that misconception arose from people applying the same numbers they’ve seen in other projects to tubby cats. If tubby cats were to use standard openzeppelin’s ERC721 implementation the wasted gas would probably be close to 5x, however by using ERC721A the percentage of total gas that is spent in tx fixed costs (eg: signature validation) is much higher, and these costs are unavoidable even if the tx is immediately rejected.

In any case, this was mistake and it would have been strictly better to revert early.

### Distribution & FUD
By far, the biggest problem that came from the gas war was psychological and mostly centered around misinformation. Imo there are two core issues at play here:
- Attention asymmetry
- Incentive to report quickly

During launch all attention is focused on the project and short tweets about it get shared a lot, and because there’s no time to get perfectly accurate data if you want to be the first to report, people just tweet their takes and a narrative is spun that might not be accurate. This is made worse because these takes are seen by a lot of people, however long analysis into these such as this post will be seen by only a few (since it requires more time to consume and attention is no longer on our project), so most people will only see the first and miss the counter-argument, thus getting an inaccurate idea.

For tubby cats, these were ideas such as:
#### "All tubbies were minted by bots"
I believe this idea came from seeing the owner distribution on etherscan and everyone complaining that their txs failed. However, after counting them with a script, only 2/3rd were minted by bots^\[1\].

#### "5000 tubbies were minted by bots and they’ll be dumped in the market, crashing price to 0"
As mentioned, only 3295 were minted by bots, and even with that, before the sale there were 15k tubbies and 10k owners, so way more than 5k tubbies had been sold before.

#### "Someone minted 950 and that made the distribution very centralized"
The first thing that the 950 bot did after mint was distribute them in batches of 50 to multiple addresses that had different activity. Thus it’s very likely that it was just a group of people that pooled funds for a bot (to pay the bot creator and reduce cost, since the more tubbies you minted the lower the price got. This is a common practice).

I wanted to examine the claims about centralization of supply, so I calculated the [gini coefficient](https://en.wikipedia.org/wiki/Gini_coefficient) for  tubby cats. This is the most used metric to track inequality in distribution: the closer it is to 1 the more centralized a distribution is, while being closer to means that it is more egalitarian. Here’s tubby’s gini coefficient compared to other wealth distributions:
| Distribution | Gini Coefficient  |
|---|---|
| Tubbies | 0.419 |
| BAYC | 0.337 |
| Ethereum| 0.63 |
| CryptoPunks| 0.7991 |
| United States| 0.852 |
| Germany| 0.816 |
| World| 0.885 |

Comparisons against world’s and countries wealth distributions are not apples to apples since a person could have tubbies in multiple addresses but I’d expect that in wealth surveys they would only count each person once. However, the comparisons make it clear: tubbies’ distribution is more decentralized than ETH and cryptopunks but more centralized than BAYC^\[2\].

Furthermore, most of those tubbies have been sold by now, however due to attention asymmetry most people are not aware of that and still think there’s someone out there with 950 tubbies.

Something that has also been quite interesting has been people’s interpretation of the gas war. What happened here was that there was a redistribution of wealth away from the team and into miners and botters, since instead of having the botters put tubbies on sale for 0.4 eth we could have had the team put them on sale for 0.5 eth directly. This means that the team could have earned an extra 0.4 eth per tubby, but that value was instead earned by bots and miners.

However, from the user’s point of view, the only difference is that in one case you can buy a tubby for 0.4 and in the other you can buy it for 0.5, so the first option should be better, but there seems to have been a lot of resistance against buying items from bots.

## The good parts
### Whitelist participation
Whitelist participation was one of the highest in the space, we got 76.1% of our whitelisted addresses to mint an NFT, which when compared with other collections is an incredibly high number, especially considering that many of the people whitelisted didn’t have to do take any action to be whitelisted, so they were completely unaware of their whitelisted status.

Looking at airdorps in the space, such as LOOKS or ENS, which had zero cost to claim, the average participation is around 50%. And that participation is in tokens minted, a metric which is skewed given that the large whales will always mint (since the tokens they receive are significant), while there’s a long tail addresses with small payout which never claim. For tubbies every address was granted a single tubby to mint, so that bias didn’t exist here.

Here are the reasons why I believe we achieved such high participation:
- Very high awareness on crypto twitter
- Most NFT projects that got whitelisted notified their holders
- We notified people that submitted a drawing but didn’t mint by tagging them on twitter
- We airdropped NFTs on polygon to all parties that didn’t mint, and then we bid on them, trigger an email to their owners

### Progressive reveal
IMO the progressive reveal was a resounding success:
- Made the loop between buying and seeing your tubby much lower, which improved experience
- Kept attention up through the sale, as new tubbies kept being revealed, instead of just having a single peak of attention at start/reveal
- Reduced the need to have unrevealed NFTs up for sale, thus making it harder for people to get rarity sniped

## TLDR
If I could go back in time these were the things I’d change:
- Give myself more control over contract, making it more mutable
- Test minting from all addresses in WL through simulation
- If doing a gas war again, restrict mints per wallet.
- Move supply check at the start of the minting function
- I’d spend less time on discord and more on wider reach channels such as twitter or actually writing code, giving 1 on 1 attention is not worth it when short on time.


#### A note on my role in tubby cats
Since I’ve been very active publicly during the sale people seem to assume that I made a very large contribution to tubby cats and I hold a lot of decision power within the team, whoever that’s incorrect:
- Team has been building it for 6 months, I was only contacted and started working on this in february
- I’m not in the main team group chat
- I’m not a mod on discord and I’m subjected to slowmode like everyone else
- I have no access to tubby cats account
- I largely didn’t participate in financial upside from the sale (defillama received a donation that accounted for 1% of total sale)
- I didn’t work on art, generation, frontend, most decision-making… I only worked on the smart contracts

My situation is much closer to a freelance dev rather than a cofounder like everyone is assuming, so please stop asking me to make decisions on behalf of the project, I just don’t have the decision power. Also, please stop asking me to be a developer for your NFT project, I won’t do it, I only do dev work for friends.

### Notes
[1] User minting count: For this I used the tubbies that were not minted by contract

[2] Caveat: Gini coefficients were taken at different points of time, so current gini coefficients of ETH and cryptopunks are likely to be different. The reason is just that I didn’t want to spend time working on scripts to calculate gini coefficients for ETH and cryptopunks. Sources are https://dtjournal.substack.com/p/how-equal-is-the-distribution-of?utm_source=url, https://www.cylynx.io/blog/on-chain-insights-on-the-cryptocurrency-markets/ and https://en.wikipedia.org/wiki/List_of_countries_by_wealth_inequality. Calculation was done with https://shlegeris.com/gini.html.
