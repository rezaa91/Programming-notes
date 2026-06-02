# System Capacity Model

## Back-of-the-envelope Calculation

Back-of-the-envelope calculations are estimates you create using a combination of thought experiments and common performance numbers to get a good feel for which designs will meet your requirements.

Common numbers/concepts which can help...

### Power of 2

| Power | Approx Value  | Name |
|-------|---------------|------|
| 2^10  | 1 Thousand    | 1KB  |
| 2^20  | 1 Million     | 1MB  |
| 2^30  | 1 Billion     | 1GB  |
| 2^40  | 1 Trillion    | 1TB  |
| 2^50  | 1 Quadrillion | 1PB  |


## Example

Estimate Twitter's storage requirements

**Assumptions:**

* 300 million active users
* 50% of users use Twitter daily
* Users post 2 tweets per day on average
* 10% of tweets contain media (~1mb)
* Data is stored for 5 years

**Calculation:**

Daily active users = 300 million * 50% = 150 million
Media storage / day = 150 million (Daily active users) * 2 (tweets per day) * 10% (contain media) * 1mb (avg. media size) = 30TB
5 year storage = 30TB * 365 * 5 = ~55PB

## Resources

* System Design Interview
