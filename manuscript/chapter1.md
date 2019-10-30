# 1. Blockchain introduction

In this chapter we will get ourselves introduced to some definitions and examples for blockchain. We will see what properties a blockchain has, what it allows us to do and what it is good for.

I> ### Definition 1
I>
I> Cryptocurrency is a digital currency in which encryption techniques are used to regulate the generation of units of currency and verify the transfer of funds, operating independently of a central bank.

I> ### Definition 2
I>
I> Blockchain is a system in which a record of transactions made in a cryptocurrency system are maintained across several computers that are linked in a peer-to-peer network.

We will try to give an example that will serve as a motivation, as well as define what are encryption and hashing techniques and how will they help us in making our system.

## 1.1. Motivation

Let's assume that you and your friends exchange money often, for example, paying for dinner or drinks, it can be inconvenient to exchange cash all the time. One possible solution to this is that you and your friends can agree to keep records of all the bills that you have. This is what is called a ledger.

![A ledger and a set of connected friends (peers)](images/ledger.png)

I> ### Definition 3
I>
I> A ledger is a book that contains a record of transactions.

Then, at the end of every day you all sit together and use the ledger to do the calculations to settle up. If you spend more than you received, you put that money in the pot, otherwise you take that money out. Every peer in the system has a *wallet* of a kind, that resembles the balance for them. Note that we have to go through all existing records in order to do calculations[^ch1n1]. 

A problem that might appear with this kind of system is that anyone can add a transaction. For example, Bob can add a transaction where Alice pays him a few dollars, without Alice approving. We need to re-think our system such that all transactions in it are verified/signed.

For signing and verifying transactions, we will use digital signatures. For now, let's assume that anyone who adds information to the ledger also adds a signature with each record, and others have no way to modify the signature, but only to verify it.

![Our ledger now contains signatures](images/signatures.png)

However, now let's assume that Bob is keeping the ledger to himself, and everybody agreed to this. The ledger is now stored in what is called a *centralized authority*. But, if at the end of the day, say Bob has some errands to run, nobody will be able to refer to the ledger.

We need a way to decentralize the ledger, such that at any given time any of the peers can do a transaction. For the purpose of this, every peer involved will keep a copy of the ledger to themselves, and when they meet at the end of the day they will sync their ledgers.

You are connected to your friends, and so are they to you. Informally, this makes a peer-to-peer network.

I> ### Definition 4
I>
I> A peer-to-peer network is formed when two or more computers are connected to each other.

For example, when you are accessing a web page on the Internet using a browser, your browser is the "client" and the web page you're addressing is hosted by a "server". This distinction is blurred in a peer-to-peer network, so every peer is both client and server at the same time.

![A decentralized ledger](images/decentralized-ledger.png)

With this system, as the list of peers grows we might run into a problem of *trust*. When everybody meets at the end of the day, how can they believe the others that the transactions listed in their own ledgers are true? We need to modify our system to support a kind of trust.

Let's say that, for each record, we will also include a special "number" (or a hash) that will represent a kind of a *proof of work*, in that it will provide proof that the transaction is really valid. For every proof of work, the users will be given some amount as a reward.

At the end of the day, we say that we will trust the ledger who has put the most of the work in it. So if Bob has some errands to run, he can catch up the next day by trusting the rest of the peers in the network. In addition to this, we want the transactions to have an order, so let's say that every record contains a link to a previous record.

![A chain of blocks - blockchain](images/blockchain.png)

So if everybody agreed to use this ledger as a source of truth, there would be no need to exchange physical money at all. Everybody can just use the ledger to put or retrieve money to it.

## 1.2. Bitcoin

Bitcoin is the world's first cryptocurrency. In November 2008, a link to a paper authored by Satoshi Nakamoto titled "Bitcoin: A Peer-to-Peer Electronic Cash System" was published on a cryptography mailing list. Bitcoin's white paper is consisted of 9 pages, however, it is mostly theoretical explanation of the design, and as such may be a bit overwhelming to newcomers.

The bitcoin software is open source code and was released in January 2009 on SourceForge. As a result of that, anyone can clone the code and make their own blockchain, thus implementing a separate cryptocurrency. Such cryptocurrencies are usually called altcoins. The design of a bitcoin includes a decentralized network (peer-to-peer network), block (mining), blockchain, transactions, and wallets, each of which we will look in details in this book.

Although there are many cryptocurrency models and each one of them differ slightly in implementation details, the cryptocurrency we'll be building upon in this book will look pretty similar to Bitcoin (with some parts being simplified). After having read this book, readers can refer to the white paper to get the complete picture for the implementation details.

## 1.3. Example workflows

### 1.3.1. Adding block to a blockchain

TODO

### 1.3.2. Mining a block

TODO

### 1.3.3. Checking wallet balance

TODO

### 1.3.4. Sending money

TODO

## 1.4. Cryptojacking

Modern browsers powerful contain a built-in programming language javascript

So when you visit a webpage on the Internet, javascript can be told to start mining

Cryptojacking is the unauthorized use of someone else's computer to mine cryptocurrency.

Since as we've seen, miners get reward for mining, this is a motivation for cryptojacking.

TODO: Add some images

## Summary

Here's what we learned in this chapter, briefly:

1. As the most basic building block, we have a transaction record (block)
1. A block contains (among other data) transactions
1. We have a ledger that is a ordered list of all valid blocks (blockchain)
1. Every peer involved with the ledger has a wallet
1. Every record in the ledger is signed by the owner, and can be verified by the public (digital signatures)
1. The ledger is in a decentralized location, that is, everybody has their own copy of the ledger
1. Trust is based upon proof of work (mining)

[^ch1n1]: There is a way we can optimize this with so called unspent *transaction outputs* (UTXOs), which we will discuss in details later.
