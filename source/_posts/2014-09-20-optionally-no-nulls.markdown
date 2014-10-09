---
layout: post
title: "Optionally no nulls"
date: 2014-09-20 15:46:38 +0200
comments: true
categories: java
published: true
---

One of the new features incorporated into Java 8 is ** Optional **, representing the
possibility that a concept may not have value, and its clearest application is
when applied to the value returned by a method or function.

``` java
public class SubscribersRepo{
  ...
  public String findAddressByPhone(final String theNumber) {
    ... some access to a data repository
  }
  ...
}

...
    String address =
        SubscribersRepo.findAddressByPhone("+930700555555");

    if(address != null) {
       sendSomeGift(address);
    }
...

```
Two things to take into account:

  1. It is not modelized that the requested information might not exist
  2. If the information does not exist, returns null or throws an exception

The problem is with point 1, because if something is not modeled, it is obviously an exceptional case.

**Optional** allows the modelization of the possible absence of information and avoids the "exceptionality" of the situation.

``` java
public class SubscribersRepo{
  ...
  public Optional<String> findAddressByPhone(final String theNumber){
    ... some access to a data repository
  }
  ...
}

...
  SubscribersRepo.findAddressByPhone("+930700555555")
                 .ifPresent(address -> sendSomeGift(address));
...
```
Now it is clear that the information might not exist, and in any case (exists or not exists), the flow of the
program is the same, no special treatment.

<!-- more -->

This is the _cannonical_ use of Optimal in Java (returning value for a function),
which is expected to be applied in libraries when they are migrated,
but in the meantime we will keep dealing with nulls.

Using Optional is simple:

# Optional Creation


``` java
Optional<String> stringOptional =
                 Optional.of("this string");

Optional<String> stringOrEmptyOptional =
                 Optional.ofNullable(theUnknownString);
```
A feature that helps with treatment of nulls is that an Optional of null is Optional.empty(),
although it must be created with ```ofNullable```.

**Tip 1**: if you  are not sure about the origin of the variable, using ```ofNullable``` is more secure.

# Value Extraction


``` java
  public void doSomething(final Optional<Long> optionalLong){

    Long longOptional =
                    optionalLong.get();

    Long longOrElseOptional =
                    optionalLong.orElse(0L);

    Long longOrElseCalculatedOptional =
                    optionalLong
                    .orElseGet(() -> calculateFibonnacci(12L));

  }
```
<code>get</code> y ```orElse``` extract the value from the Optimal, and ```orElse```
offers a default value in the case that the Optional holds a null.

**Tip 2** : calling ```get``` will throw an NPE if the value inside the Optional is null. Extract the
value using ```orElse``` is more secure.

# Value Transformation

``` java

  ...
  Optional<Tax> stateTax = findStateTax("mo");

  Integer taxToApply =  
                stateTax
                    .map(tax -> sum(tax + federalTax))
                    .orElseGet(()-> calculateHistoricalTax());


  Optional<Discount> discountBase =
                          findDiscount("wonderfulPhone");
  Optional<Discount> discountState =
                          findDiscount("wonderfulPhone", "mo");

  Optional<Discount> fullDiscount =
                  discountBase
                  .flatMap(d -> Optional.of(d + discountState.get());

```

**Tip 3** : when a default value is needed and its calculation is expensive,
<code>orElseGet</code> is better, it will be evaluated only if it is needed.

# Conditional Execution

``` java

  Optional<Integer> price = findPrice("wonderfulPhone");

  if(!price.isPresent()){
    System.out.println("Sorry, still cannot sell it");
  }

  price.ifPresent(p -> notifyPriceToAllReservations());

  price
      .filter(p -> p > 9_000_000)
      .ifPresent(p -> rememberItIsVeryWonderful());
```

**Tip 4**: the less _funcional_ of all these methods is _isPresent_, if possible go with the others.


Just one more thing about Optimal: Java 8 is the first approach of Java to the Functional Programming,
and the main target has been the addition of lambdas to the language, but in the case of Optional,
it is a [Monad](https://gist.github.com/ms-tg/7420496).
