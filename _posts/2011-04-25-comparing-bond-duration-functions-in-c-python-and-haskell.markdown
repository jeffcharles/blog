---
title: Comparing Bond Duration Functions in C, Python, and Haskell
---
Lately I've been studying for a Working Capital Management exam coming up on Wednesday, April 25th. As you can probably guess, the subject matter can be dry at times so I've tried to come up with something productive to do during study breaks. Since it's been a few weeks since I've coded something I decided to see how implementing a bond duration calculation would compare between C, Python, and Haskell.

For those who don't know, C is one of the older, more traditional statically typed imperative programming languages that has widely influenced most newer languages like Java and Python. Python is a newer language that mixes imperative and functional constructs and uses dynamic typing. Finally, Haskell is a statically typed functional language.

In terms of the calculation being implemented, <a title="Bond Duration" href="http://en.wikipedia.org/wiki/Bond_duration" target="_blank">bond duration</a> is a weighted average time until repayment as well as a measure of how sensitive a bond's price is to changes in interest rates. The calculation takes a coupon rate (the percentage of face value paid out on an annual basis), the current interest rate (typically this would be the prevailing rate on a one year American treasury bill), and a time to maturity (how many years until the bond matures). Coupon rates and interest rates are negatively related to duration while time to maturity is positively related. You can check out the Wikipedia article linked above to see the formula for bond duration.

Here is my C implementation for bond duration:

{% highlight c %}
#include <math.h>
#include <stdio.h>
#include <stdlib.h>
/* Calculates bond duration
   Preconditions:
       coupon (i.e., coupon rate)  >= 0
       interest_rate >= 0
       time_to_maturity > 0
*/
double duration(double coupon, double interest_rate, int time_to_maturity) {
    double weighted_pv = 0;
    double bond_price = 0;
    double payment_pv;
    int n;
    for(n=1; n <= time_to_maturity-1; n++) {
        payment_pv = coupon / pow((1 + interest_rate), n);
        weighted_pv += n * payment_pv;
        bond_price += payment_pv;
    }
    // principal repayment
    payment_pv = (1 + coupon) / pow((1 + interest_rate), time_to_maturity);
    weighted_pv += time_to_maturity * payment_pv;
    bond_price += payment_pv;
    return weighted_pv / bond_price;
}
int main(int argc, char** argv) {
    printf("%.2f\n", duration(strtod(argv[1], NULL), strtod(argv[2], NULL),
        atoi(argv[3])));
    return 0;
}
{% endhighlight %}

This is one possible C implementation. I tried to implement it in such a way that it would perform faster than the Python or Haskell implementation by taking advantage of C's imperative nature. For example, it will only run through the loop once and it only uses as much memory as it needs to. There is a small repetition in calculating the payment present values for principal repayment, the only difference being that the principal repayment adds 1 to the coupon rate. Granted that this could have been handled inside the loop but as it's different logic and it always affects only the last payment term, I think it makes more sense to place it after the loop and have a tiny DRY violation.

Now, here is my Python implementation:

{% highlight python %}
import operator, sys
def duration(coupon_rate, interest_rate, time_to_maturity):
    """
    Calculates bond duration
    Preconditions:
        coupon_rate - (float >= 0) e.g., 0.04
        interest_rate - (float >= 0) e.g., 0.08
        time_to_maturity - (int > 0) e.g., 5
    """
    payments = [coupon_rate for _ in range(time_to_maturity)]
    payments[time_to_maturity-1] += 1 # principal repayment
    discount_rates = [(1 + interest_rate) ** n
        for n in range(1, time_to_maturity+1)]
    payments_pvs = map(operator.div, payments, discount_rates)
    weighted_pvs = map(operator.mul, range(1, time_to_maturity + 1),
        payments_pvs)
    return sum(weighted_pvs) &#47; sum(payments_pvs)
if __name__ == "__main__":
    print "%.2f" % duration(float(sys.argv[1]), float(sys.argv[2]),
        int(sys.argv[3]))
{% endhighlight %}

Now, I could have coded this almost identically to the C version but that would've been boring and pointless. Instead I chose to take advantage of Python's functional language constructs, specifically list generators, map, and reduce to illustrate a different coding style. As opposed to trying to do the calculations in one go, this approach builds lists of payments, discounts, and weighting co-efficients and then "zips" them together (think of it like a zipper joining two teeth together) using map before summing up the weighted present values and bond payment present values using reduce.

Finally, here is my Haskell implementation:

<pre lang="haskell">
import System.Environment
import Text.Printf
-- Calculates bond duration given a coupon rate, interest rate, and time to
-- maturity
--
-- Args:
--   c - coupon rate (>= 0), e.g., 0.04
--   k - interest rate (>= 0), e.g., 0.08
--   t - time to maturity (> 0), e.g., 5
duration :: Double -> Double -> Int -> Double
duration c k t = (sum weighted_pvs) &#47; (sum payments_pvs)
    where weighted_pvs  = zipWith (*) (map fromIntegral [1..t]) payments_pvs
          payments_pvs  = zipWith (&#47;) payments discounts
          payments = replicate (t-1) c ++ [1 + c]
          discounts = [ (1 + k) ^ i | i <- [1..t]]
main = do
    (c:k:t:_) <- getArgs
    printf "%.2f\n" $ duration (read c :: Double) (read k :: Double)
        (read t :: Int)
</pre>

This, interestingly enough, is coded almost identically to the Python version above in terms of how it goes about solving the problem. The big difference is in the syntax of the two languages. I'm not going to go into too much detail in this blog as there are plenty of other resources out there if you're interested in finding out more about Haskell. I just find it illustrative of the idea of using a functional language to describe solving a problem without having to worry about loops and local variables and the other minutia in imperative languages (important as they are). This code captures the essence of a solution without distracting the reader with the specifics of how the solution is achieved (perhaps with the exception of the map fromIntegral call that's required for zipping weighting co-efficients).

In conclusion, I hope this post helps to illustrate some simple differences between imperative and functional coding styles using a simple problem. In the future, I might also try coding a bond duration implementation in Scala and Clojure, which are a couple languages on my list to learn. If you have any suggestions for how to improve the code or this post, please leave a comment.
