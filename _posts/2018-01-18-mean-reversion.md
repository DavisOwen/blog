---
layout: post
title: Mean Reversion Strategies
categories: [finance]
tags: [finance]
description: Creating a Stationary Time Series and Hedge Ratio Problems
---
Strap in boys and gals, it's gonna be a long one.

If you want to skip some intro on some basic mean reversion strategy concepts like adf, johansen test and such you can skip right down to the star delimited area below.

Mean reversion is the concept that a time series will tend to revert to its mean (average) over-time. This means that if you notice the price of an asset is 2 standard deviations above its 50 day moving average, it would be a relatively smart idea to short sell. This is the basic idea behind a strategy like Bollinger Bands.

However, this is not always a good idea. In practice, asset prices tend to trend upwards (or downwards), and there is typically some sort of random walk involved. Therefore, we have a dilemma. Clearly, mean reversion strategies can be very powerful, granted we can be relatively confident that the asset price is not trending. We could long and short the asset indefinitely, cashing in on the ups and downs automatically using a simple algorithm. How can we tell if an asset is trending or mean reverting (also known as stationary)?

The augmented dickey fuller test is a statistical test that test exactly this question. I implemented it using the statsmodels tsa (time series analysis) library. Typically the ADF test is introduced as a means for testing pairs trading strategies. A pair trade strategy is created by running an ordinary least squares regression on the time series of two separate assets. The beta (or slope) is used as the hedge ratio, one of the main topics of this post. In pairs trading, the hedge ratio has a simple real life interpretation. A ratio of 0.5 means that for every $1 increase in the price of asset A, asset B will increase by 0.5$ (and vice versa).

Using this hedge ratio (beta) we can create a residual graph of the two asset's time series. In this example, this means that we subtract 0.5 times the price of asset B from asset A. Theoretically, this should form a stationary time series. You want to test that mathematically? Good, thinkin bud! Too bad I'm too dumb to pedantically explain the ADF test. Thankfully, it doesn't really matter.

Running the ADF test gives significance values at the 10%, 5%, and 1% levels. If your test statistic is above the 1% level, you're pretty damn confident you got a mean reverting time series.

Enter: Me. I wanted to do a naive test of the pair trades of all the combinations of stocks listed in the S&P 500. This is pretty easy to implement, just use beautiful soup to scrape the wikipedia page for all the names of S&P 500 stocks, and use the quandl's free WIKI dataset API to download your time series data on the fly. From there, simply run the ADF test on all the different combinations of pair trades and find the most significantly stationary time series. There's also something called the Hurst Exponent that you can optimize, but I found it's not super important.

I ran backtests using a simple bollinger bands strategy, and I was not happy with the results. Large drawdowns, low sharpe ratio, mediocre returns. This is also because I personally try to run strategies with the most amount of data possible, to assure that the stationarity of the portfolio isn't simply a one or two year phenomenon. This larger time frame leads to more difficulty in finding working strategies, but when I do find it, I'll be pretty confident in its predictive power.

To put it frank, I was devastated. And quite frankly confused. I could see with my own two eyes that the time series were stationary. WHY wouldn't a strategy like bollinger bands work? Especially if I took the mean and standard deviations of the entirety of the historical data in the backtest time frame (yes I was cheating) as opposed to the N-day moving average mean and standard deviations, one would expect a pretty good increase in performance. Wrong.

What was going on here?

I thought maybe it wasn't actually stationary enough.

Enter: the Johansen test.

JoJo, my BOY. A true baller. Exactly what I was looking for. If two assets moving in tandem wasn't adequate to create a stationary portfolio, I thought maybe I could create a stationary portfolio. Now to my knowledge, this is something that finance firms do all the time. The more price movements you add into a portfolio, the less volatile, and more predictable the price movements. However, it is more useful to have a rigorous statistical test that will tell you for sure if your portfolio is truly stationary.

The Johansen test is a statistical test that will not only tell you if a certain basket of assets that is larger than 2 can form a stationary portfolio, it will also tell you exactly what the hedge ratios are in order to form this stationary portfolio. I had to scour the web to find a good implementation in python of the johansen test, and I can give it to anyone who asks in the comments. I found it in a random branch on the statsmodels tsa github page.

Anyway, I went through the same process as with the ADF test, except I decided to run create a 11 asset large portfolio. I picked 11 because I wanted to choose 1 asset from each of the 11 sectors listed in the S&P 500. Even with the simplification, it is still obviously a very algorithmically complex script.

1 man

11 nested for loops

1 destiny

Actually not too bad though because you can use parallel processing to cut down a bit on time. Also I found after running for like 30 minutes that almost all of the portfolio's had ADF test statistics more than twice the amount that I was getting for the simple 2 asset pair trading time series.

I picked a portfolio I liked and rolled with it.

Now I took my Bollinger Pair strategy I backtested, and added in the 11 asset portfolio. I named this file bpj.py, aka bollinger pair johansen.
You can find it in my github.

The results? Again slightly disappointing. Maybe I had too high expectation with this, but something just didn't seem right with these results.

 Results

Enter: obnoxious debugging with no guarantee of progress.

I refuse to accept these results. The theory is quite simple. I can see that this portfolio is, historically, guaranteed to return to the mean after straying a couple standard deviations from the mean. Therefore, why is the backtest giving subpar (under s&p) level performance? It shouldn't be commission, as there are not that many trades happening. I debugged using get_open_orders and context.portfolio.positions to see if my order were going through. Some of them were taking months to complete? What? I ended up having to write an adjustHedgeRatio function that would scale down the percent of each stock to order so that all the orders could complete in 1 day, to be consistent with the 0.025 volume slippage model. However, that was largely unnecessary as I realized the real issue was that I had been using a custom bundle that was for some reason dividing the volume by 1000 and therefore creating unnaturally low daily volume data for the backtester. The custom bundle was also giving me trouble as the data was not split and dividend adjusted. It created a random spike of like 200% on a random day by not accounting for the split in the number of shares I owned. Anyway, moving back to the default quantopian-quandl bundle fixed this. Even after all this, the results were still bad.

Back to the drawing board.

I honed in on what I now believe to be the true source of error in this strategy, that seems so trivial to me but I have managed to overthink it to the point of insanity.

Enter: the Hedge Ratio

It might actually be a fundamental misunderstanding on my part. The Hedge Ratio for the Johansen test of this particular portfolio is an eigenvector composed of 11 values:

[ 0.02598021 -0.1192674 -0.01062298 -0.79423715 0.44579013 0.1413708 -0.27075362 0.08992664 0.07375943 0.29200028 -0.51136716]

Now, the most astute of you will right off the bat recognize that the absolute value of the sum of these do no add to 1. Is it necessary that it is normalized? Idk, so I tried normalizing it and of course it made no difference, in fact made the results worse.

The thing is, I know that these numbers are correct, because if you use them as coefficients in the 11 portfolio time series, it creates the stationary time series we are looking for. The problem is how they are implemented in the backtest, I believe.

Expected Time Series

I use order_target_percent, and use each of the numbers as a raw percentage of the portfolio value. Again, the fact that these percentages add up to more than 1 is a little unsettling. However, I don't mind hypothetically going into debt in the backtesting world because hypothetically I'm a baller.

In order to really test what the problem is, I decided to just make simple algorithm that just buys the portfolio on day 1 of the backtest and holds it for the entirety. Then, I can compare the graphs of portfolio returns and portfolio values to the graph of the expected time series. The results? Not even close.



Queue twilight zone music.

This whole order_target_percent thing isn't working. Some theories of mine are this:

Maybe, instead of ordering a target percentage of the whole portfolio value, the assets values have to be percentages with respect to each other. i.e., some sort of order_target_percent with respect to the context.portfolio.position_values. Maybe the one asset you buy the most of will be the price that you normalize all the other hedge ratios with respect to.

That honestly doesn't even make sense to me either ^

My other theory that I tried out was this. If I compared the hedge ratio's every day to the ACTUAL percentage ownership of each asset with respect to the portfolio value, I found that at the beginning they were approximately close, but over time the percentages changes. Obviously, because the portfolio will either go up or down in value as more money is made or lost. However, the ratios of all the stocks with respect to each other should remain unchanged. Regardless, I decided to do a strategy where I rebalance the portfolio every day to make sure that I continue to keep my portfolio weights consistent with the hedge ratios, as seen here.

 

The left colum is the actual weights, and the right column is the target hedge ratios, and each block is one day. Compared to earlier where the portfolio is only bought once on the first day.



In a last ditch effort, I decided I would implement gradient descent to minimize the squared difference between the expected hedge ratios and the actual hedge ratios, altering the number of shares purchased of each stock. Honestly, I was just psyched to actually be using the calculus I've learned in college for something seemingly useful. That code can be found in bpj.py as order_target_portfolio_percentages. I played around with the learning rate and initial conditions a bit, but was never able to get the cost function to really get below 0.1, without starting to get to ordering billions of shares of stock.

It ended up being a failure, and I even if it were to have worked, I was still altering the number of shares bought of each stock by a non integer number.

Clearly, you can't own a fractional share of a stock. You can only own discrete integer values of stock. Therefore, there is a certain limit to the precision of percentages you can own of stocks, unless you start buying billions of shares of each stock.

If you've read this far, I posted this on the website quantopian about 6 months ago and just copy and pasted it. I don't want to revisit it very much because the ultimate conclusion was that, while this was a good exercise, the strategy is untradeable in real life. I actually was able to re mediate the problems I had above and was able to use a hedge ratio that gave great (phenomenal) backtest results when the johansen test was run on the entirety of the dataset that was also used for backtesting. However, in reality no one has this luxury.

After trying to use train-test splits of the dataset to see if one could extract a hedge ratio that would work into the future to predict prices well, I had no luck. The returns were abysmal. The problem also lies with using rolling averages. You can overfit with the lookback window, and also even optimizing the lookback window parameter using something like bayesian optimization did not lead to good results. The fact is that the means change drastically as you go from data point to datapoint, and you get very unpredictable buy and sell points. If you could look into the future and draw a straight line through the time series, and buy above or below that line, it would work perfectly. But that line cannot be predicted with rolling averages. Maybe it could be possible with 1000's of millions of years of historical data, but obviously we don't have that.
