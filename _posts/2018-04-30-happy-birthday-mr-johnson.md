---
layout: post
title:  "Happy Birthday, Mr. Johnson!"
date:   2018-04-30 11:00:00 +0300
image: /assets/placeholder.png
categories: development testing
comments: true
---

This happened long time ago when I was supporting production deployment of a big enterprise system of a bank.

The deployment was not automated. Special people with production access were doing necessary changes. I, as a developer, was telling them via Skype what to do. The people were not familiar with the system. They were not developers even. Due to such an approach the deployment usually took very long time. It was already late evening when this happened.

Suddenly, a Jenkins job that was doing a deployment of a component became red. That was an unpleasant surprise as the component was the last one in the long list of components. I opened the log and found ouy that the unit tests failed. They expected 20 € as a price of an insurance, but the actual price was 25 €. That was really weird: nobody touched the component for a month and all the previous builds were green.

Frankly, the first idea that came to my mind was to ignore unit tests and continue deployment. After all, that's how developers usually deal with failing unit tests, don't they? However, given a criticality of this component, that was not a good idea. The component was responsible for calculation of insurances prices. People can forgive broker UI layout. But they never forgive any issues with their money. So I had only one choice: to fix the test as soon as possible. I started to dig into the error.

OK, I thought, yesterday everything was fine. Today the tests fail. This means that either the code still changed. Or the service was taking some data (probably some price coefficients) from the outside (that can be either another service or a database) and the unit tests were written in a wrong way, i.e., they relied on the data from the outer world.

I started with the first option. I checked the commits history, the history of the job itself, nothing changed since last month. Then I took the green build and the build that failed and compared their SHA1 hashes. They were identical. This meant that nothing changed, definitely.

Then I switched to the second option. It was a bit trickier as the service itself was the legacy, with thousands of lines of spaghetti code per class. I was looking for all signs of that the service consumes something from the outer world. JDBC connection strings, JNDI bindings, REST clients configurations, SFTP to external file system... Nothing. The only external RESTful service that provided the coefficients for the price (to be able to change it without changing the code) was properly mocked in the unit tests.

The only way that left after this is to dig into the price calculation business logic. That was the place that was written long time ago and nobody really wanted to touch it. After some time of futile attempts to understand the logic my attention was attracted by the following line:

```java
int age = DateUtil.countAgeUsingPersonalCode("280752-7918")
```

The insurance became more expensive than the test expected. The insurances usually become more expensive when the clients get older. Wait a second...

Today is July, 29th. Mr. Johnson (that was the name of the fictional client from the unit tests) celebrated his sixtieth birthday yesterday. And sixty years is one of the frontiers when the price for the insurance gets higher.

Now everything was completely clear to me. The code didn't change indeed. And the unit tests didn't consume any services from the outer world: the only outer service was properly mocked. However, one external thing they still "consumed" was time. Yesterday, and for all the earlier time, the tests were passing because Mr. Johnson was still young and was eligible for cheaper insurance. Starting from today, however, he turned 60, the price for him got higher, while the test still expected it to stay the same...

### The Morality Of The Story

Never allow your unit tests depend on _anything_ external. Time is also the external thing, though it may seem stable and predictable.

One good "test" for unit tests can be changing your PC clock to some very strange time, like to a distant past or future, and check that the tests keep passing and do not raise false-positive errors.