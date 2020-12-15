The Fundsmith Equity Fund is an Open Ended Investment Company (OEIC) managed by Terry Smith.  When it was launched in November 2010, Terry Smith commented that the investment industry in the UK served only itself, and that his fund was focused on delivering superior returns to shareholders.  From inception to December 2020 the fund returned an average compound annual rate of 18.2%, beating the FTSE100, S&P500 and MSCI World Index.

![Fundsmith Performance vs FTSE100, S&P500, MSCI World Index](/images/FS_performance.png)

(Please note : I will use 'holdings' to describe the stock equity within the fund. Others may use 'stocks', 'equity', 'contents')
The fund does not disclose the weights of the holdings, nor a breakdown of all the holdings in the fund.  I wanted to see if it was possible to pry open the fund and use some data science techniques to predict the weights of the holdings. The process was broken down in to the following steps:

1. Scrape data from the [Fundsmith monthly factsheets](https://www.fundsmith.co.uk/fund-factsheet) (scroll the bottom of the page) to find when holdings were bought and sold.
2. Use the [Yahoo Finance python plugin](https://pypi.org/project/yfinance/) (aka 'yfinance') to download the daily price for every holding in the fund, as well as the fund price.
3. Don't forget to get the currency exchange rates (also from yfinance)
4. Use linear regression with holding prices as predictors and fund price as target to find the line of best fit. The co-efficients describe the amount of influence each holding price has on the fund price movement, and in theory should align with the actual holding weight.
5. BONUS : Use the model to predict the fund price on a test set of data.
6. BONUS : I thought I would try some ARIMA modelling to illustrate that 'past performance does not predic future returns'
