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
  <img src=https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/pdfplumber_train_raw.png alt="Raw training results from pdfplumber">
  
  After a bit of RegEx I wrote a function that extracted the relevant informaion out of the above screenshot, as seen below.
  <img src=https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/pdfplumber_train_results.png alt="Results from pdfplumber after RegEx">
  
  However, when I applied this to all the other pdfs, the results were not as I thought they would be.
   <img src=https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/pdfplumber_actual.png alt="Results from pdfplumber and RegEx applied to all pdfs">
   
   The lesson learned is to ensure your test data is in the same format as your training data. In this case, there were 2 issues:
   1. The team at Fundsmith hadn't decided on a strict format early on in the funds existence. As such, the placement of the relevant information would move around the pdf from month to month. It eventually settled into a strict format after 8 months.
   2. The size of the 'comment' section impacted the placement of the top 5 contributors and top 5 detractors.  If a lot had occurred that month, then the top 5s would be shifted down and fall outside of the parameters specified in pdfplumber.
  </p>
</details>
 Using an online pdf-to-text converter, I created one text file for each month's pdf commentary. I opened the files, read them and extracted the relevant information using RegEx, storing them in a dictionary.
 
 ![Extracting the relevant information from monthyl PDFs](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/extract_pdf_info.png)

From here, I created a Pandas DataFrame and input the information from the July 2019 semi-annual report detailing the full breakdown of the holdings.  From there, I went through the comments to see when each holding was bought or sold and input this in to the DataFrame (with '1' indicating the holding is present, and a NaN for when the holding is not).

<details>
 <summary><b>Solving the challenge of missing holdings</b></summary>
 <p>3 holdings were specified in the 2019 semi-annual report that were not found in the bought/sold part of the comment section of the pdfs. These need to be accounted for, so what approach did I take to find them?
  Without a definite month where Fundsmith commented on the buy/sell of these 3 holdings, I decided to search all the pdfs in their entirety for the first mention of each of the holdings.  Possible issues that need to be kept in mind are that the holdings may have been bought, but are only mentioned in the monthly pdf when it reaches one of the top 10 holdings, or is one of the top 5 contributors or top 5 detractors.  The following code shows my approach and the result.
   <img src=https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/missing_holdings.png alt="Searching pdfs for any mention of Amadeus, Diageo and Intertek">
 </p>
</details>
