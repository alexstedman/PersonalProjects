# Predicting the Fundsmith Equity Fund

The Fundsmith Equity Fund is an Open Ended Investment Company (OEIC) managed by Terry Smith.  When it was launched in November 2010, Terry Smith commented that the [investment industry in the UK served only itself](https://www.theguardian.com/money/2010/nov/06/investment-funds-terry-smith-warren-buffet), and that his fund was focused on delivering superior returns to shareholders.  From inception to December 2020 the fund returned an average compound annual rate of 18.2%, beating the FTSE100, S&P500 and MSCI World Index.

![Fundsmith Performance vs FTSE100, S&P500, MSCI World Index](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/FS_performance.png)

(Please note : I will use 'holdings' to describe the stock equity within the fund. Others may use 'stocks', 'equity', 'contents')
The fund aims to own no more than 30 high-quality holdings, and aims to own those holdings for as long as possible to allow the company to compound in value. These 2 factord make it a good candidate for applying some data science techniques to.  The fund does not disclose the weights of the holdings, nor a breakdown of all the holdings in the fund. It does, however, provide a commentary every month on any outright sales and purchases made within any partcular month, as well as the top 5 contributors and top 5 detractors for that month and the top 10 holdings. I wanted to see if it was possible to pry open the fund and use some data science techniques to predict the weights of the holdings. The process was broken down in to the following steps:

1. Scrape data from the [Fundsmith monthly factsheets](https://www.fundsmith.co.uk/fund-factsheet) (scroll the bottom of the page) to find when holdings were bought and sold.
2. Use the [Yahoo Finance python plugin](https://pypi.org/project/yfinance/) (aka 'yfinance') to download the daily price for every holding in the fund, as well as the fund price.
3. Don't forget to get the currency exchange rates (also from yfinance)
4. Use linear regression with holding prices as predictors and fund price as target to find the line of best fit. The co-efficients describe the amount of influence each holding price has on the fund price movement, and in theory should align with the actual holding weight.
5. BONUS : Use the model to predict the fund price on a test set of data.
6. BONUS : I thought I would try some ARIMA modelling to illustrate that 'past performance does not predic future returns'

## 1. Scraping the Data
The first step in getting the data on the fund holdings was to find a document or webpage that described the *entire* holdings of the fund.  I did a thorough search of the Fundsmith website and found exactly that in the semi-annual report of July 2019 on the European version of the Fundsmith website (pages 11-12 [here](https://www.fundsmith.co.uk/docs/default-source/annual-reports-and-audited-financial-statements/unaudited-semi-annual-report-for-the-period-from-1-january-2019-to-30-june-2019.pdf?sfvrsn=4)).  This was to be my starting point.

Inspecting the Fundsmith webpage showed the links to all the monthly pdfs containing the commentary I wanted on the purchases/sales made within the month.
![The Fundsmith Equity Fund Documents page with web inspector](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/FS_page_inspection.png)
I used [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/) to scrape the links to all pdfs and saved them to file.

<details>
 <summary><b>Challenges in scraping pdf links and how I tackled them</b></summary>
 <p>The links weren't always of the same format. Some ended with 'sfvrsn=4', others 'sfvrsn=6'. Some had date format yyyy-mm, others were mm-yyyy. I got round this with a simple nested for loop.  It's not an amazing solution, I admit, but it is easy to read for a programmer with no knowledge of the project.
  
<img src="https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/notebook_scraping_links.png" alt="Code for scraping pdf links"></p>
</details>

The next step was to use the requests library in python to open each link and write the contents to file.
With the pdfs downloaded, I then had to find a way to extract the relevant information from them.

<details>
 <summary><b>Failures on extracting information from pdfs, and what I learned</b></summary>
 <p>I first attempted to extract the information using <a href="https://pypi.org/project/pdfplumber/">pdfplumber</a>, a tool that converts a pdf to text line-by-line (column formatting is ignored). I specified a text box from where pdfplumber could extract the information from, with the results below.
  </p>
</details>
