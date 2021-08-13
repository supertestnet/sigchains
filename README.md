# Sigchains
A sigchain is a chain of timestamped signatures. It can be used to issue and transfer tokens at the speed of the lightning network.

Suppose an oracle maintains a publicly viewable array -- which I will call a sigchain -- of entries consisting of pubkeys, messages, timestamps, ordinal numbers, and signatures. Suppose an entry on the sigchain has this form: ["pubkey","message","timestamp","ordinal","signature"], and suppose the sigchain has this form: [entry,entry,entry,etc. ]. Suppose the signature in each entry on this sigchain is only valid for the pubkey within that entry concatenated with the message, the timestamp, and the ordinal number. Supposing such a sigchain exists and at least one person -- the oracle -- can add entries to it, imagine a world where users send data to the oracle consisting of a pubkey and a message, which I will call a key:message pair. If users send their key:message pair to the oracle, the oracle can -- possibly in return for a lightning payment -- create a timestamp and an ordinal number, concatenate the pubkey, message, timestamp, and ordinal number, sign the result, and immediately add a new entry to the sigchain in the format mentioned above. Any user who submits a key:message pair can then verify that his or her message is attested on the sigchain at a certain timestamp. An ordinal number is a numerical value which follows a timestamp on a sigchain. If there are multiple entries in a sigchain with the same timestamp, those entries are supposed to be ordered alphabetically, and the ordinal number attests to their order for a given timestamp.

Suppose a user called Alice wants to send a message to a user called Bob which is verifiably attested on the sigchain. Alice can create a pubkey and a message, sign her message with her pubkey, and send the oracle a lightning payment plus her pubkey and her message. If the oracle accepts the payment and adds an entry to the sigchain, Alice can send a copy of her key:message pair to Bob. Bob can verify that the pubkey given to him by Alice really signed the message given to him by Alice or discard it if not. Supposing it did sign it, Bob can verify whether or not that message is attested on the sigchain by using Alice's pubkey to look up the set of entries in the sigchain containing Alice's pubkey. If Bob finds a set of entries containing at least one match, Bob can treat the entry in the set with the earliest timestamp as the only valid entry for that pubkey and verify that the signature for that entry is valid for Alice's message or discard it if not.

In this way, Alice can quickly send Bob arbitrary messages which are verifiably attested on the sigchain.

# Censorship

If a group of people wish to use a sigchain without fear of censorship, it seems wise to keep the oracle who runs the sigchain in the dark as much as possible about the contents of their messages. Otherwise, the oracle might censor their entries and refuse to add them to the sigchain -- even for a fee -- if it does not want to attest to certain content.

In the sigchain system described in this protocol, censorship is unlikely if users do not reveal the contents of their messages to the oracle, but only send it a hash of their messages. As long as users do not reveal their messages to the oracle, it only sees key:hash pairs and a payment. It is blind to the messages it is attesting to. Therefore, assuming that users only send the oracle a key:hash pair but not a full message, the oracle should not have any reason to treat any messages differently from others. Moreover, other users can still verify that the oracle attested to a user's full message if the message creator provides them with the full message. The message recipient can hash the message, check if that hash is attested on the sigchain, and -- relying on the assumption that there are no hash collisions for ripemd160 hashes -- infer that the oracle's attestation is valid for the full message and only that message, nothing else.

However, this procedure is not sufficient to avoid all censorship of the sigchain. For one thing, users might reveal the contents of their messages to the oracle, such as if the oracle is -- perhaps unbeknownst to them -- one of the people to whom they are sending a message. Also, the oracle might decide to censor an entry not based on the content of the associated message but based on the pubkey which was sent alongside the entry. This might happen if oracles must, for legal reasons, abide by a blacklist of pubkeys and not attest to messages created by those pubkeys. Also, supposing the oracle ceases operations at some point, that is equivalent to censoring all of its users if no one can add to the sigchain anymore.

# Censorship resistance

To ensure that sigchains are censorship resistant despite these obstacles, there is an alternative way for users to add entries to a sigchain. Specifically, users can take any message that they want to put on the sigchain and create one or more bitcoin transactions where their message is submitted via one or more op_return fields (if the message is so long that it takes up multiple op_return fields, other users may parse these op_return fields serially). Suppose a user encodes their op_return data in this way: ["pubkey","message","sigchain id"]. The sigchain id is a ripemd160 hash of the oracle's pubkey. The oracle must publicly disclose their pubkey so that users can verify its signatures on the sigchain. Any user can therefore hash the oracle's pubkey to ascertain the sigchain id. The use of sigchain ids also allows for many sigchains to exist with different oracles or quorums of oracles.

In sigchain entries created via op_returns, the pubkey field is 32 bytes, the sigchain id is 20 bytes, and the message can use up the remaining 28 bytes in the op_return (and possibly additional op_returns as desired by the user). A set of one or more op_return entries can substitute for getting an entry attested by the oracle. It is noteworthy that op_return entries are not necessarily fully open to public view. A user who wishes to keep everyone blind to the contents of their op_return entries may choose to only add a hash of their message to the sigchain and then later choose whether or not to disclose to some people the full message which hashes to that value.

# Independent sigchain reconstruction

Suppose that a sigchain's entries are all shared among peers in a manner similar to how bitcoin transactions and blocks are shared on the bitcoin network. Users who wish to independently construct a sigchain and come to consensus on its contents could do so using a gossip protocol. A sigchain entry is always considered a valid addition to the chain if and only if it matches the format of entries signed by the oracle or the format of entries created via op_returns. Another condition is that the signature in oracle entries must come from the oracle's pubkey and be valid only for a concatenation of the pubkey, the message, the timestamp, and the ordinal number. Also, a consensus rule of the sigchain is that all copies must sort every entry based on its timestamp. The entry with the earliest timestamp is the first entry on the sigchain, followed by the next earliest, and the next earliest, and so forth. If some entries have the same timestamp, those entries can be sorted alphabetically in ascending order and assigned an ordinal number: the one with the lowest alphabetical value is first and gets the ordinal number 0, followed by the one with the second lowest alphabetical value which gets the ordinal number 1, and so forth.

It may be noted that op_return entries don't have a timestamp in the text of the op_return. For these entries, a timestamp can be inferred using the timestamp of the block the message is included in. This eliminates the need to put a timestamp in the op_return itself. The oracle id can substitute for the oracle's signaure in such messages because the oracle is not attesting to this message, the blockchain is, and all the user needs to know is which sigchain the message belongs to, which can be ascertained using the sigchain id. Programs which independently construct the sigchain and participate in the gossip protocol -- which I will call sigchain nodes -- construct the sigchain by entering in all of two types of entries: oracle entries which look like ["pubkey","message","timestamp","ordinal","signature"] and op_return entries which look like ["pubkey","message","timestamp","ordinal","sigchain id"].

Sigchain nodes independently construct the sigchain by simultaneously scanning bitcoin's blockchain for entries in that format and the oracle's array of entries. They consider both sets of entries -- sorted by timestamps and assigned ordinal numbers as necessary -- as constitutive of the sigchain. As long as the sigchain is defined not only as the set of entries in the oracle's array but also as the set of entries constructed using the data in op_returns on bitcoin's blockchain, the oracle cannot censor the sigchain but only deny users access to its own timestamping service. Effectively, there are two sets of people who can add entries to the sigchain: the oracle and bitcoin's miners. Either set can be "hired" to perform this service. If the oracle becomes censorious or goes offline, bitcoin's miners can be relied upon to be less censorious, albeit possibly more expensive or slower.

# Immutability

There may be cases where sigchain wallets discourage pubkey reuse by checking if a pubkey has signed multiple messages and considering only one of them valid, specifically the one whose timestamp is the earliest. In such a case, it might seem possible to trick users into accepting a message which is later invalidated when a second entry signed by the same pubkey is inserted into the sigchain by the oracle or by bitcoin's miners with a timestamp before the original. This could be especially devastating if monetary value in the form of tokens are considered sent and confirmed only if they are in a message which has the earliest timestamp for that pubkey in the sigchain.

## Post-dating

One scenario to guard against is where bitcoin miners mine a block which adds an entry to the sigchain where the timestamp of the new block is in the past. In this context, it is useful to recall that there are limitations about how old a block's timestamp can realistically be. A block is invalid if its timestamp is older than the median timestamp of the past 11 blocks. In practice, no block has been mined whose timestamp was older than 7000 seconds in the past, which is a bit less than 2 hours. As a result, there is an easy way to reduce miners's ability to past-date a sigchain entry: if a sigchain entry appears in a bitcoin block, sigchain nodes can add 2 hours to the timestamp of that block and use the adjusted timestamp for the sigchain entry. This procedure should be perfectly reproducible by all sigchain nodes and relatively harmless to users. Users are not expected to use the op_return method of adding entries to the sigchain except in extraordinary circumstances, so they should not be too harmed by the fact that the op_return method is considerably slower. In fact, the op_return method is already expected to be the slower option because op_returns aren't added to bitcoin's blockchain until mined, whereas the oracle can add entries to the sigchain immediately. So who cares if op_return entries are even slower due to being post-dated by 2 hours?

## Merge mining

Preventing oracles from past-dating an entry in the sigchain is harder. I do not know a good way to absolutely prevent it in all cases. However, the oracle's ability to past-date entries can be limited. Suppose that every time any entry is added to the sigchain, either via op_returns or via the oracle, the oracle must -- before adding any other entries -- use its own pubkey to create an entry on the sigchain which I will call a merge-minable entry. The message in this merge-minable entry must be a hash of several hashes concatenated together, specifically, a hash of its previous entry, if such a previous entry exists, and a hash of each entry that made it onto the sigchain since the oracle's previous entry if such a previous entry exists. All of these entry hashes must be ordered by the timestamps of the corresponding entries, or, in the case of multiple entries with the same timestamp, ordered alphabetically as described earlier.

If the oracle creates a merge-minable entry every time any entry is added to the sigchain, then most of the entries on the sigchain will belong to the oracle's pubkey. There will basically be an entry originating with the oracle immediately following every entry that does not originate with the oracle. In cases where several entries are added to the sigchain via op_returns in one block, the oracle must also hash these entries and include them in proper order when it creates the merge-minable entry.

Merge-minable entries are useful because they can prevent the oracle from altering the history of the sigchain without detection. Moreover, some of the data in these entries can be added to bitcoin's blockchain through a blind merge mining transaction. If this is done, anyone who obtains the full list of sigchain entries via the sigchain's peer to peer network can verify the history of the sigchain up to the time of the latest merge-minable entry whose signature has been added to bitcoin's blockchain in an op_return.

The merge mining procedure will be described in a moment. First I want to mention what merge minable entries do. An oracle might attempt to alter the history of a sigchain in three ways: by changing the timestamp of a past entry, by inserting a new entry in between the timestamps of two older entries, and by dropping an old entry altogether. Users who already have a full copy of the sigchain can detect these alterations of the sigchain's history because their copy of the sigchain will no longer match the one the oracle presents on its website. But -- without merge minable transactions -- users who do not have a full copy of the sigchain, such as spv wallet users and new node users who are still syncing the sigchain, won't know that insertions or alterations are invalid. With merge minable entries, past entries in the statechain can be used to independently construct what the hash of the merge minable entry should be. If, for a given timestamp:ordinal pair, a user sees two different merge-minable entries signed by the oracle, that can then detect that some alteration has taken place and the oracle has violated some rule. Merge-minable entries thus make alterations of a sigchain's history detectable by any user.

What's more, they are called merge-minable entries because their signatures can be added to bitcoin's blockchain in an op_return by anyone. If that is done, then the earliest oracle signature that is valid for a merge minable transaction at a given timestamp:ordinal pair can be considered the only valid entry for that timestamp:ordinal pair. Users can thus come to consensus about the order and history of a sigchain up to the time of the latest merge-minable transaction whose signature has been added to bitcoin's blockchain.

# Token issuance

Messages and sigchains can be used to issue and transfer assets at speeds roughly equivalent to bitcoin's lightning network. Suppose a user called Alice wants to issue 100 tokens. She can create an issuance message, hash it, sign the hash, and add her key:hash pair to the sigchain either via the oracle or via an op_return. An issuance message must have this format: "pubkey 1ef...50a received 100 tokens from an issuance transaction, the token id is 5d2...9a4." The recipient pubkey must sign issuance transactions and the token id must be globally unique for each asset.

Once Alice's signature is in the sigchain, Alice can prove to anyone that she issued 100 tokens by sending them her message and her pubkey. Other people can use the information provided by Alice and the sigchain to verify that 100 tokens were in fact issued and were sent to a pubkey. Alice could also issue more tokens of the same kind by creating a similar message again, thus inflating the supply of tokens. If her token is one where inflation is desirable, such as a usd-based stablecoin where more tokens are periodically issued when users pay to create them, that is well and good.

In cases where token issuances should be publicly auditable, there is a way to do that. The issuing pubkey can ensure that it only adds one publicly readable message to the sigchain containing the terms of the issuance. If a company issues these tokens, it may put on its website which pubkeys are valid issuers of its tokens, and user wallets can ensure that they only accept tokens with provenance that traces back to a public issuance by one of those approved pubkeys.

Later, I will explain how users who receive tokens can verify the provenance of those tokens before providing a receipt for the transaction. Users who desire to reject tokens whose provenance does not lead back to a public issuance can use wallets which refuse to provide a receipt for such transactions. Thus client side validation with sigchains is sufficient for ensuring that token issuance is auditable in cases where it is desirable for a token to have auditable supply.

# Token transfers

Suppose a user called Alice controls the pubkey 1ef...50a which has already received 100 tokens from an issuance transaction or another transfer transaction. Alice wants to transfer 50 tokens to another pubkey. She can create a transfer message, sign it, hash it, and add her key:hash pair to the sigchain either via the oracle or via an op_return. A transfer message must have this format: "pubkey 4a1...990 received 50 tokens from pubkey 1ef...50a, the token id is 5d2...9a4." The sender's pubkey must sign transfer transactions. Clients only see them as valid if the sender sent tokens they previously received either via issuance transactions or through other transfers.

# Proof of provenance

Suppose a user called Alice wants to transfer 50 tokens to a user called Bob. Alice gets a pubkey from Bob (his pubkey is 4a1...990) and uses one of her own pubkeys (1ef...50a) to sign a transfer message. Then she sends Bob a copy of her pubkey, 1ef..50a, along with a copy of her transfer message. The transfer message is just like the one mentioned above: "pubkey 4a1...990 received 50 tokens from pubkey 1ef...50a, the token id is 5d2...9a4," where pubkey 4a1...990 is Bob's pubkey. Bob can use Alice's pubkey to look up the set of entries in the sigchain containing Alice's partial pubkey and her signature. If Bob finds a set of entries containing at least one match, Bob can extract the full pubkey from the signature of each entry in the returned set and verify whether the pubkey matches the one Alice sent to Bob. If it does not match, he can discard that message and look for the next one, because it is not signed by Alice's pubkey, and therefore it is just as irrelevant as the other messages in the sigchain. (The pubkey extracted from a signature might not always match the partial pubkey prefixed to the signature for several reasons, one of which is that malicious actors can pay to add fake signatures to the sigchain prefixed with the partial pubkey of a user they don't like. It is important to check if the pubkey extracted from the signature matches the one provided to you by your counterparty lest you reject good transactions because of malicious actors.)

If none of the messages in the sigchain are signed by Alice's pubkey, Bob can refuse to give Alice a receipt for the transaction because she did not send him the tokens he wants -- she gave him a pubkey which never signed a transfer message or never got it attested on the sigchain. If, however, one of the signatures returned by his search "checks out," Bob can check if more than one checks out. If more than one checks out, Bob can check which one's timestamp is earliest and consider only that one to be a valid message. If every sigchain wallet abides by this rule, that should prevent double spends.

If some wallets do not abide by this rule, the users of those wallets will not have that protection against double spends. This major downside may encourage them to adopt wallets that only treat the earliest timestamped message by a single pubkey as valid.

Assuming Bob's wallet verifies that the earliest timestamped message from Alice's pubkey "checks out" in terms of having a signature that is attested on the sigchain, Bob's wallet is now at the part of the transaction where it should check the provenance of the tokens transferred to him. In order to do this, Alice's wallet must send Bob's wallet proof of provenance. Ideally, this should be done at the same time the token transfer message is sent. Perhaps Alice received her tokens directly from an issuance transaction; if so, she should send Bob the message which transferred her the tokens and the pubkey of the token issuer -- that is her proof of provenance.

Bob's wallet can use Alice's proof of provenance to check if the issuance transaction follows the rules his wallet follows. For example, was the issuance signed by a public key belonging to a company he trusts to legitimately issue tokens? Is the signature of the issuance attested to on the sigchain? Was it a publicly auditable issuance (assuming he doesn't want to accept tokens where the issuance was done privately and is therefore not auditable)? If the proof of provenance fails any of these checks, Bob's wallet can refuse to provide a receipt.

Perhaps Alice did not receive her tokens directly from an issuance transaction but from a prior transfer. If so, her proof of provenance will be longer. She will need to send Bob a trail of transfer messages and pubkeys leading back to the token issuance -- which could be a rather long proof of provenance. Once Bob's wallet has a complete proof of provenance from Alice, it can check everything out by ensuring that the tranfers and issuance transaction are attested on the sigchain. If any check fails, Bob's wallet can refuse to provide a receipt. In this way, Alice can only send Bob money and expect a receipt if she first validates that their wallets follow the same rules (such as by checking version numbers) and that her proofs of provenance check out before sending anything to the sigchain for attestation. This does impose a storage burden on Alice, who must store a train of pubkeys and messages in order to validly send anyone money and expect a receipt. But these storage costs should be low as long as pubkeys and signed messages are small.

Sigchain users can always see if a given pubkey ever double signed any messages on the sigchain and only accept the earliest messages from a given pubkey. They can also use the sigchain plus the proof of provenance to ensure that they are not receiving counterfeit tokens but only tokens that come from sources they trust. These protections should reduce the likelihood of counterfeiting and double spending to acceptable levels of trust minimization. All a user needs to do is verify that the sender has enough tokens to cover the expenditure (that is what the proof of provenance is for) and that they aren't accepting a late message (by only considering the earliest message signed by Alice's pubkey as valid). Given those two provable facts -- the sender has enough tokens to cover the expenditure and isn't double spending anything in this transaction (because this pubkey never signed anything before) -- there is no significant risk to the recipient.

# Token Change

In many of these examples I assumed Alice received 100 tokens at one of her pubkeys and later sent out 50. If wallets will never consider a second expenditure from her pubkey as valid, how can she ever get her money back? The answer is similar to how bitcoin works: when Alice sends money to Bob, she can send any "change" -- leftover funds not given to Bob -- to a new pubkey belonging to herself. Thus her transfer transaction would look like this: "pubkey 4a1...990 received 50 tokens from pubkey 1ef...50a and pubkey 30c...bb4 received 50 tokens from 1ef...50a, the token id is 5d2...9a4," where 4a1...990 is one of Bob's pubkeys and 30c...bb4 is one of Alice's. If Alice is concerned that such a transaction will give Bob a way to check if she still has that money in the future (since he will see the transaction too), Alice can immediately send her money out again to yet another new pubkey. Since these transfers use lightning to pay the transfer fee and Bob will not receive this second transfer message, it should be a cheap way to restore Alice's privacy.

# Lightning speed

Issuing tokens or transferring tokens should not typically involve much waiting. The steps for issuing tokens are typically: validate some data, send it to an oracle, and pay a lightning payment. The issuance should ordinarily be considered valid as soon as the oracle adds the entry to the sigchain, which shouldn't take more than a few seconds since the oracle doesn't have to validate anything. Transferring tokens should also be very fast. It is the same procedure except now you have to also send a message to the recipient and wait for him to give you a receipt (after he looks up some signatures on the sigchain and then validates the data). Again, none of this should take a long time.

One exceptional case might be this: suppose an entry to the sigchain is made via an op_return instead of via the oracle. This might be necessary for a user whose pubkey is on some sort of blacklist. Hopefully, to avoid delays when buying their coffee, blacklisted individuals would transfer the tokens to a new pubkey via an op_return transaction well before they go to get coffee. These transactions are uncensorable since they are done using bitcoin's blockchain and, unless the blacklisted person leaks the new pubkey to someone and it gets to the oracle, they should be able to use their funds again censor-free (and therefore at lightning speed). If they don't take that precaution, waiting for a confirmation will probably be necessary unless the recipient is okay with receiving unconfirmed payments on the blockchain.

It is important to note that the sigchain oracle can alter the past of a sigchain except for messages that happened before the most recent merge mined entry. Some users therefore may not consider a transaction finalized until a merge mined entry from a time after their transaction has been added to bitcoin's blockchain. I don't think most users will think this way unless their transaction is of a kind where their counterparty is likely to have an opportunity to collude with the oracle to alter the sigchain's history. I doubt this will usually be a concern, especially for low value transactions.

One might wonder if another exceptional case involves doublespend attempts. For example, suppose Alice adds an entry to the sigchain via the oracle which sends some tokens to Bob and simultaneously adds an entry via an op_return which sends the same tokens back to herself. Might Bob have to wait in this case? No. If Bob's wallet sees an entry in the sigchain via the oracle and another entry possibly made via an op_return (regardless of whether or not it is confirmed), all Bob's wallet needs to do is rely on the policy of "only the earliest one is valid." Op_return entries are post-dated by 2 hours, so as long as the oracle entry is confirmed now, that is the one Bob's wallet should consider valid.

It is also possible that the oracle is colluding with Alice and intends to drop Bob's entry and replace it with Alice's, or wait for the op_return to do that. This is a potential concern for Bob, who cannot know that this will happen in advance. If that happens, he will probably give a receipt and lose an item of value without getting any money for it, but at least he will have a proof of fraud by the oracle in the form of a state update that was signed by the oracle and then removed from the sigchain. Bob can publish this proof of fraud on the internet to dissuade people from using the dishonest oracle. If the oracle can be trusted not to act maliciously in this way, the system is robust. 

# Smart contracts

Token transfer messages and token issuance messages can specify spending conditions that must be followed by token recipients. A field can be added to the transfer message saying something like: "The recipient can only spend this if they sign the transfer and disclose the preimage to this hash: 7ea...026, alternatively the sender can spend it if the timestamp of the expenditure is after time such-and-such." That would effectively put the funds in an htlc on the sigchain. A network of payment channels using htlcs could then be created that is basically a clone of the lightning network. Other spending conditions could also be enforced, developers could let their imaginations go wild.

If smart contracts are used, recipient wallets will need to understand the spending conditions of the tokens they receive and know what to do with them. If a wallet receives tokens but does not understand the spending conditions, it should refuse to provide a receipt. Perhaps the user can download a wallet which does recognize the spending conditions, import his private keys to that wallet, and then use the new wallet to issue a receipt.

A wallet will also need to understand the spending conditions of the ancestors of its tokens in case they restrict their descendants in ways which are not mentioned in the latest message. They can use the proof of provenance to check if any ancestor transactions created spending conditions which the current wallet does not understand.

# Death of an Oracle

There may be circumstances where an oracle can no longer provide its services and must stop operations. If a wallet only knows about the oracle's attestations by checking the oracle's website, that wallet's users could be in for a bumpy ride if it ceases operations cold-turkey. Their wallet would look up data on the sigchain via bitcoin and via the oracle's website, which, if it no longer exists, contains no transactions. Thus the transactions on bitcoin's blockchain would suddenly be the only valid ones, which could result in double spends where oracle transactions that used to exist are suddenly no longer considered part of the sigchain for such wallets.

This scenario seems similar to a scenario where some bitcoin users use wallet which rely on a node that stops working. Their wallet also stops working and consequently it can seem like they lost money. But users who run their own sigchain nodes would not be affected in the same way, nor would users who get data about the sigchain not only from the oracle's website but also from the peer to peer network.

There are other downsides to a sigchain's death, including that the sigchain can not only be added to by means of bitcoin transactions, slowing everything down and probably making everything more expensive. Strategies to mitigate or avoid the downsides of an oracle's death include: first, choose oracles responsibly. Don't use a sigchain run by a disreputable oracle who might go away at any moment. Consider using a sigchain maintained by a quorum of oracles who split the income obtained via transaction fees. Reliable oracles should also post all updates to the sigchain in multiple places simultaneously so that users' wallets don't display errors due to a website being down, and wallets should get data from a peer to peer network rather than only relying on the sigchain's website.

An oracle's death is sort of a "censor everyone" scenario. No one can add to the sigchain by means of a dead oracle, but since sigchains are censorship resistant, users can still add to it by adding transactions to bitcoin's blockchain. This would make sigchains as slow as bitcoin after the oracle's death, but, if there are sigchain wallets with lightning support, users could still migrate to those wallets and basically get the sigchain up to speed again, albeit with some new encumbrances.

Another option to mitigate oracle death is to hard fork to a new oracle and ensure the new oracle has a copy of the sigchain. If the oracle's death is not sudden but prepared for ahead of time, this even brings the possibility of a smooth transition.
