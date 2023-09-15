Sponsor can fund in any ERC20 token but the winnings are distributed from only a particular ERC20 token address.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/Distributor.sol#L92

## Summary
- Here, we are allowing sponsors to fund in any token and that token can be any token, either whitelisted or not. So there is a need to allow the sponsor to only fund in the whitelisted token.
- Also, even if the sponsor funds in any whitelisted token but at the end when we distribute price we are using only a particular token which is distributed to the winners and all the other whitelisted tokens in which the sponsor funded will remain unutilized. So, this can lead to accumulation of token funded by organizer in the proxy. There is a method which can only be used by owner to distribute the wrongly sent sponsored funds to their respective owners but it is irrelevant, why accept the tokens if they are not whitelisted. (Prevention is better than cure)

## Vulnerability Details
This is high vulnerability as it can lead to wastage of funds sent by organizer.
Suppose that there were 2 tokens, USDC and USDT as whitelisted. The sponsors funded in both USDC and USDT tokens.
But when we distribute tokens to the winners, we are only using a particular token (say the organizer chose USDC) and as a result of which the other USDT token will remain unutilized and as a result of which the owner of protocol will call the ```distributeByOwner``` and 5% of token were taken by stadium.

So, if N tokens are whitelisted but at the end when we distribute winnings the organizer will choose only a particular token from all of the N tokens, and the N-1 tokens will remain unutilized. And the owner will call the ```distributeByOwner``` function N-1 times (if the sponsors funded the amount in all tokens) to give back their amount. As a result of which 5% of the total respective token amount is taken every time to recover the unutilized sponsored funds.

Either the organizers already informs the sponsor that at the end they will distribute the tokens in only x token, so that they don't fund in any other token even if the other token was whitelisted.

## Impact
High impact on our protocol.

## Tools Used
Manual Testing

## Recommendations
- The organizer should set the token for which they want to distribute prizes to the winners, and only allow fundings by sponsors for that particular token only.
- But if we say that we will distribute prizes to winners in some rounds on the basis of the tokens in which we received the funding then it can lead to injustice among the winners and not a fair distribution of the prizes.
- So, to solve this we can deploy the Proxy contract at the starting of contest and define a function in the implementation which will manage the fundings received by sponsors, and will only allow that whitelisted token selected by the organizer.