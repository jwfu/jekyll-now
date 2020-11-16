---
layout: post
title: Stock Total Return Calculator
---

Here in the UK, large ETFs are usually offer both accumulating and distributing dividend treatments.  Over time, dividend reinvestment has a significant contribution to total return, as shown below for the S&P500:

![_config.yml]({{ site.baseurl }}/images/spyTotalReturn.png)

I couldn't to find an easy way to determine total return for arbitrary stocks, so I wrote a small tool does this using [AlphaVantage through Pandas Datareader](https://pandas-datareader.readthedocs.io/en/latest/readers/alphavantage.html).

The tool is available on PyPI using `pip install gainz-jwf`.  The code is hosted in [Github](https://github.com/jwfu/gainz).