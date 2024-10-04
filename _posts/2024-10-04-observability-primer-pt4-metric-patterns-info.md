---
title: Metric Patterns
date: 2024-10-04 04:00:00 +0000
categories: [Blogging]
tags: [software,architecture]
image:
  path: /assets/img/common/patterns.jpg
  alt: patterns
---

A useful pattern is the `_info` pattern, a common prometheus antipattern is to pollute the labelspace of a metric with a lot of redundant, unchanging label patterns. this can lead to a lot of issues with the underlying platform, which we may explore a little later in this series.

For example, we have a metric that tracks user sign ups by a few labels.
```
user_sign_up_count{"country_code"="UK","region"="europe",country="United Kingdom",browser="Chrome"} 10
```

We could break this down into two metrics to reduce our storage footprint, and increase performance

```
user_sign_up_count{"country_code"="UK",browser="Chrome"} 10
country_info{"country_code"="UK","region"="europe",country="United Kingdom"} 1

```

And we could reconstitute that label using a PromQL Query like so

```
user_sign_up_count{browser="Chrome"} 
* on(country_code) 
  group_left(region, country) country_info
```

As such, we're able to drastically reduce our storage footprint, while still maintaining the same level of granularity in our data

## A practical Example

One example is this issue i raised back in April 2023 for the Ethereum Prysm Client

https://github.com/prysmaticlabs/prysm/issues/12348

My use-case for this was to retrieve the valiator index natively from the native prysm client metrics, the aim here was so that we'd be able to join to other metrics using the pubkey to retrieve the index for the purposes of enriching the label set.

Lots of tools make use of the validator index as the primary key for determining which validator you're referring to, one example is the beaconchain browser `https://beaconcha.in/validator/${index}`
`
Wouldn't it be nice to be able to simply link this? 

For this to work, we'd need to add a label to the validator index to provide us with the validator index

Which leads us to this Pull Request

https://github.com/prysmaticlabs/prysm/pull/14473

Simply put, we're adding a new label to the existing validator status metric, to the end of being able to join with other metrics to retrieve the index, or to be used in dashboard links for further exploration e.g. https://beaconcha.in/validator/1111111 


This has been added to the already existing `validator_statuses` metric so that
- We don't have a redundant info metric & the associated cardinality with a custom `info` metric
- We make a non-breaking change to the metric contract for the existing `validator_statuses` metric by adding an additional label


## In the next post! 

Given the content of this post, we start to get deeper into the realm of prometheus, as a "metric series" begins to stray into Prometheus Territory, the main difference here is that OpenMetrics defines the wire format, and Prometheus provides the query engine and overarching storage and aggregation platform, which we'll explore a bit more in the next post
