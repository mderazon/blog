+++
date = "16 Apr 2017"
title = "Generate a random order-number"
draft = false
url = "/posts/generate-random-order-number"
tags = ["node", "ecommerce"]
image = ""
comments = true
share = true
menu= ""
author = "Michael DeRazon"
+++

For one of my projects, I needed to generate a random order number that will be sent to customers in an email after they order. Lots of companies do that, it usually looks like this:


### Requirements
1. Has to be all digits.
2. Needs to be unique. We don't want two orders to have the same order number.
3. Has to be random - i.e. not enumerable. We don't want the orders to be consecutive for example (5001, 5002, 5003, ...) so that people can't guess other orders numbers.

### Format
Order number is *not* the same as order id that you use internally to uniquely identify your orders. For order ids, it's best to use a known format like [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) or something similar.

There's no standard format for order numbers. They should be long enough to be able to hold enough orders but at the same time not too long as people have to use them to identify an order and sometimes read them out loud to a customer support representative.
It's also good practice to divide them into couple of sections for better readability.

I chose a 4-6-4 format (x is a digit):
```
xxxx-xxxxxx-xxxx
```
There wasn't really any particular reason for this choice other than nicely symmetric. This gives us 14 characters to identify an order which should be plenty enough for most cases.

### Solution #1 - Generate and validate
The first approach for this problem is:
1. Generate a random 14 digits number.
2. Search for this number in database to see if we had it before (unique).
3. If we found it, try another one. If it's unique, use it.

This is an obvious approach and makes sure that we don't have collisions.
Problem is that we make a round-trip to the database and if the db is hosted somewhere else the call would involve the network and it can be slow. If we're making the call to the db anyway chances are that in most cases we won't have to make an extra call due to collision but we still have to add code path to handle the case of collision.

Can we remove the need to make a call to validate ?

### Solution #2 - Generate a universally unique number
How can we generate a number that we'll know is unique without having to validate ?

We use time 🕒

System time is easily accessible in every programming language and time is progressive which means that if we read it in one moment and then read it again a moment later, we are promised to get a different value.

A good approach will be to use the [Unix epoch time](https://en.wikipedia.org/wiki/Unix_time) format which returns a 13 digits integer (at least until year 2286 when it'll become 14 digits).
To get to 14, we can just add one random digit as padding.
The number is guaranteed to be unique down to the resolution of a millisecond which is great and also the padding digit add a little extra safety for two orders that happened at the same time.

so, in Javascript we can do:
``` js
function orderNumber() {
  let now = Date.now().toString() // '1492341545873'
  // pad with extra random digit
  now += now + Math.floor(Math.random() * 10)
  // format
  return  [now.slice(0, 4), now.slice(4, 10), now.slice(10, 14)].join('-')
}
```

Problem with this approach is that this number is not really random (excluding the padding digit) and we don't want to expose our sophisticated order numbering scheme :-)

### Solution #3 - Generate a universally unique and obfuscated number
The trick here is to use encryption to encrypt the epoch time:

1. Get current time and add extra digit.
2. Encrypt the result to obfuscate the original number and make it look random.

If we use a symmetric-key encryption (encrypt and decrypt with same key) we preserve the uniqueness property of the number after encrypting since it's always possible to decrypt it back to the original number.

We need to use an encryption algorithm that yields a numbers-only ciphertext. I have read about [format-preserving encryption](https://en.wikipedia.org/wiki/Format-preserving_encryption):
> In cryptography, format-preserving encryption (FPE) refers to encrypting in such a way that the output (the ciphertext) is in the same format as the input (the plaintext).

Meaning that if we encrypt a 14 digits number the result will also be a 14 digits number.

I have opted to use a simple FPE technique from a "[prefix cipher](https://en.wikipedia.org/wiki/Format-preserving_encryption#FPE_from_a_prefix_cipher)".
It's a pretty straightforward method and consists of:

1. Take all digits [0..9] and encrypt each one of them however you want. I use `aes-256-cbc` with the password `michael` in this example:

| digit |            encrypted             |
|-------|----------------------------------|
|     0 | cc542e3575b66e7e18da4efea5bd5dd0 |
|     1 | 05dc4db56eecf0bbe86ecd5b32b0bd8a |
|     2 | bb6aa9a0e4459e44dd1161f8151bf5b8 |
|     3 | a4113f6a28d807ec67864284c4c8fa7f |
|     4 | 136dbce2a64017352c433586441b961f |
|     5 | cf3fc8ac69a41f1ac43d7ba37e7fb68c |
|     6 | 7c154f41a8bbf39a844ffb8cce82e08f |
|     7 | 736197baa4f81c7ec5de42f099e84902 |
|     8 | 5068ecec92f52719c904361c9b91b929 |
|     9 | 393d6b7d2eddcf2021600a744406fa2c |

2. Sort the encrypted results:

| index | digit |            encrypted             |
|-------|-------|----------------------------------|
|     0 |     1 | 05dc4db56eecf0bbe86ecd5b32b0bd8a |
|     1 |     4 | 136dbce2a64017352c433586441b961f |
|     2 |     9 | 393d6b7d2eddcf2021600a744406fa2c |
|     3 |     8 | 5068ecec92f52719c904361c9b91b929 |
|     4 |     7 | 736197baa4f81c7ec5de42f099e84902 |
|     5 |     6 | 7c154f41a8bbf39a844ffb8cce82e08f |
|     6 |     3 | a4113f6a28d807ec67864284c4c8fa7f |
|     7 |     2 | bb6aa9a0e4459e44dd1161f8151bf5b8 |
|     8 |     0 | cc542e3575b66e7e18da4efea5bd5dd0 |
|     9 |     5 | cf3fc8ac69a41f1ac43d7ba37e7fb68c |

3. We now have a cipher method to encrypt every digit:

```
F(0) = 1
F(1) = 4
F(2) = 9
.
.
```

I have created a small node library that uses this method to encrypt digits or other things. Check out the complete code if you want: [node-fpe](https://github.com/mderazon/node-fpe).

### Summary
To wrap things up, here are the steps to generate a unique randomly looking order number:
1. Get current Unix time (ms).
2. Add another random digit to complete 14 digits.
3. Encrypt with FPE
4. Format (4-6-4)

That's all.
Code is available in my Github repo: https://github.com/mderazon/order-id

### Collision probability
The order number is not guaranteed to be unique. Two orders will have the same order number if they both happened in the same millisecond and also both have the same padding digit.

Let's divide this event into two separate probabilities and then multiply them to get the final probability:
1. P(two orders at the same ms)
2. P(two orders with the same padding digit)

#### Probability of two orders at the same millisecond
We are assuming orders are independent of each other and can come at any time during the day. This means they are controlled by a [Poisson process](https://en.wikipedia.org/wiki/Poisson_distribution) with a parameter λ that gives the expected number of events per unit time.

Assuming we have 1,000,000 orders per day, λ = 1e6/8.64e7 (events per ms) = (events/day)*(days/ms).

In a Poisson process, the time between successive events (let's call it the inter-event interval) has an [exponential distribution](https://en.wikipedia.org/wiki/Exponential_distribution) with mean 1/λ. In our example, this gives an average of 86.4ms between orders.

Given that an event has just occurred at time t0, we want to calculate the probability that the next event occurs at time t1 ≤ t0+1 ms. That is, the inter-event interval will be between 0 and 1ms. To do that, we can integrate the probability density function (PDF) of the inter-event interval from 0 to 1ms. This is the same as evaluating its cumulative distribution function (CDF) at 1ms. The CDF of the exponential distribution is:
```
1−e^(−λt)
```
Evaluating this at λ=1e6/8.64e7 and t=1ms gives a probability of ~0.0115

#### Probability of two orders with the same padding digit
This one is easy, probability of choosing the same number twice is just 1/10.

#### Overall probability
Assuming 1,000,000 orders per day and a random padding number, the probability comes down to a simple multiplication:
```
P(two orders with the same number) = 0.0115 * 0.1 = 0.00115
```
Or roughly 0.1%
1/99999999999999
