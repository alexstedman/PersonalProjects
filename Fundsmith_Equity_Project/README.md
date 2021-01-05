# Predicting the Fundsmith Equity Fund

The Fundsmith Equity Fund is an Open Ended Investment Company (OEIC) managed by Terry Smith.  When it was launched in November 2010, Terry Smith commented that the [investment industry in the UK served only itself](https://www.theguardian.com/money/2010/nov/06/investment-funds-terry-smith-warren-buffet), and that his fund was focused on delivering superior returns to shareholders.  From inception to December 2020 the fund returned an average compound annual rate of 18.2%, beating the FTSE100, S&P500 and MSCI World Index.

![Fundsmith Performance vs FTSE100, S&P500, MSCI World Index](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/FS_performance.png)

(Please note : I will use 'holdings' to describe the stock equity within the fund. Others may use 'stocks', 'equity', 'contents')
The fund aims to own no more than 30 high-quality holdings, and aims to own those holdings for as long as possible to allow the company to compound in value. These 2 factors make it a good candidate for applying some data science techniques to.  The fund does not disclose the weights of the holdings, nor a breakdown of all the holdings in the fund. It does, however, provide a commentary every month on any outright sales and purchases made within any partcular month, as well as the top 5 contributors and top 5 detractors for that month and the top 10 holdings. I wanted to see if it was possible to pry open the fund and use some data science techniques to predict the weights of the holdings. The process was broken down in to the following steps:

1. Scrape data from the [Fundsmith monthly factsheets](https://www.fundsmith.co.uk/fund-factsheet) (scroll the bottom of the page) to find when holdings were bought and sold.
2. Use the [Yahoo Finance python plugin](https://pypi.org/project/yfinance/) (aka 'yfinance') to download the daily price for every holding in the fund, as well as the fund price. Don't forget to get the currency exchange rates (also from yfinance)
3. Explore the data
4. Use linear regression with holding prices as predictors and fund price as target to find the line of best fit. The co-efficients describe the amount of influence each holding price has on the fund price movement, and in theory should align with the actual holding weight.
5. BONUS : I thought I would try some seasonal modelling using Facebook's Prophet model to illustrate that 'past performance does not predict future returns'

## 1. Scraping the Data
The first step in getting the data on the fund holdings was to find a document or webpage that described the *entire* holdings of the fund.  I did a thorough search of the Fundsmith website and found exactly that in the semi-annual report of July 2019 on the European version of the Fundsmith website (pages 11-12 [here](https://www.fundsmith.co.uk/docs/default-source/annual-reports-and-audited-financial-statements/unaudited-semi-annual-report-for-the-period-from-1-january-2019-to-30-june-2019.pdf?sfvrsn=4)).  This was to be my starting point.

Inspecting the Fundsmith webpage showed the links to all the monthly pdfs containing the commentary I wanted on the purchases/sales made within the month.

![The Fundsmith Equity Fund Documents page with web inspector](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/FS_page_inspection.png)
I used [Beautiful Soup](https://www.crummy.com/software/BeautifulSoup/) to scrape the links to all pdfs and saved them to file.

----
<details>
 <summary><b>Challenges in scraping pdf links and how I tackled them</b></summary>
 <p>
  The links weren't always of the same format. Some ended with 'sfvrsn=4', others 'sfvrsn=6'. Some had date format yyyy-mm, others were mm-yyyy. I got round this with a simple nested for loop.  It's not an amazing solution, I admit, but it is easy to read for a programmer with no knowledge of the project.
  
<img src="https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/notebook_scraping_links.png" alt="Code for scraping pdf links"></p>
</details>

----
The next step was to use the requests library in python to open each link and write the contents to file.
With the pdfs downloaded, I then had to find a way to extract the relevant information from them.

----
<details>
 <summary><b>Failures on extracting information from pdfs, and what I learned</b></summary>
 <p>
  I first attempted to extract the information using <a href="https://pypi.org/project/pdfplumber/">pdfplumber</a>, a tool that converts a pdf to text line-by-line (column formatting is ignored). I specified a text box from where pdfplumber could extract the information from, with the results below.
  
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

----
 Using an online pdf-to-text converter, I created one text file for each month's pdf commentary. I opened the files, read them and extracted the relevant information using RegEx, storing them in a dictionary.
 
![Extracting the relevant information from monthyl PDFs](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/extract_pdf_info.png)

From here, I created a Pandas DataFrame (as a Time Series) and input the information from the July 2019 semi-annual report detailing the full breakdown of the holdings.  From there, I went through the comments to see when each holding was bought or sold and input this in to the DataFrame (with '1' indicating the holding is present, and a NaN for when the holding is not).

----
<details>
 <summary><b>Solving the challenge of missing holdings</b></summary>
 <p>
  3 holdings were specified in the 2019 semi-annual report that were not found in the bought/sold part of the comment section of the pdfs. These need to be accounted for, so what approach did I take to find them?
  
  Without a definite month where Fundsmith commented on the buy/sell of the missing 3 holdings, I decided to search all the pdfs in their entirety for the first mention of each.  Possible issues that should be kept in mind are that the holdings may have been bought, but are only mentioned in the monthly pdf when it reaches one of the top 10 holdings, or is one of the top 5 contributors or top 5 detractors.  The following code shows my approach and the result.
   
   <img src=https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/missing_holdings.png alt="Searching pdfs for any mention of Amadeus, Diageo and Intertek">
 </p>
</details>

----
After accounting for all the holdings, I interpolated the data from monthly to daily (business days) resulting in a DataFrame with observations on UK business days from November 2010 to November 2020, and columns of every holding the fund has ever had, with its presence in the fund indicated by a '1' or NaN if not present.

## 2. Use yfinance to import stock prices and exchange rates
The next phase of data scraping was to find the prices of the holdings on the days they were in the fund.  For this, I used yahoo finance (which I will refer to as yfinance from now on).  This is a simple python plugin that allows you to search for publically traded companies and returns information on the opening price, closing price, intra-day lows and highs, any stocks splits that occured that day and what dividends were paid. What a great little tool!

yfinance uses company ticker symbols as input, and I didn't have these.  No worries though; I called the column names of the DataBase and then manually searched the internet for them. I put these in a list and then iterated over it with a simple for loop, placing the symbols in to the yfinance input variable.  The dates I chose for it to return were the full span of the funds existence - November 2010 to November 2020.  I chose this way over extracting the dates from the DataFrame because I could simply multiply the results by the 1s and NaNs in the DataFrame to filter them down to their relevant dates that way.

![yfinance initial results](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/yfinance_result.png)

Uh-oh.  3 holdings couldn't be found by yfinance on the public stock exchange; Sigma-Aldritch (ticker: SIAL), CR Bard (ticker: BCR) and Dr. Pepper Snapple (DPS).

The prices for these holdings needed to be found another way.  I scoured the internet and found them through investing.com. It was then a small step to copy the data from the website, input it into a csv file and read them into the DataFrame.

For reference, at this point my DataFrame has columns of every holding that has existed in the Fund, with '1's and NaNs for dates where the holding is present or not in the Fund (I shall call this these 'visibility').  There are also columns of every holding with its opening price on the day.  There is one final bit of information to obtain to complete the data collecion - currency movements.  The Fundsmith Equity Fund invests in stocks in the UK, USA and Europe.  yfinance can again be used to pull the currency exchange rates.  Once this was done, the final DataFrame was created by multiplying the visibility of the holdings by the opening prince of the holdings and (if necessary) the exchange rate, followed by appending the Equity Fund price (again, drawn from yfinance).

## 3. Explore the data

Let's have a look at the data.  Below is a plot containing the price movement of each holding on the x-axis, the fund price movement on the y-axis, and colour-graded by time in the fund (red is when the holding entered the fund, through to orange and yellow, ending in blue when the gholding either exited the fund or is up to the end of the time series in November 2020).

![Holding prices vs Fund price](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/holding_vs_fund.png)

One of the first things you'll notice is the absence of data for Del Monte, Nike and Starbucks.  The latter 2 entered the fund only at the end of the time series.  They were holdings that were being built up and hadn't formally reached the desired weighting (by Fundsmith's own account) but they were mentioned in the monthly commentary which is how they had come to be included in the DataFrame.  Del Monte is a more complicated story involving its parent company with a very similar name buying Del Monte, and then changing its name back to Del Monte.  No data can be found, as far as I'm aware, on the share price of the original Del Monte company.

The correlation between holding price and Fund price is greater than 90% for 28 holdings, and greater than 80% for 39 holdings (out of a total of 45 holdings, excluding Del Monte, Starbucks and Nike).  This is looking promising.

## 4. Use Linear Regression to predict weights of the fund holdings

I just want to re-iterate the hypothesis I'm testing.  By knowing the prices of the underling holdings over a significant period of time, the price of the Fund can be calculated.  Performing linear regression (using the holding prices as predictors and Fund price as target) trains the model with coefficients assigned to each holding.  The coefficients are a 'weight' quantifying the influence of each holding's price on the Fund price.  In theory, the coefficients should then be the same as the true weights of the holdings in the fund.

Before performing the regression I needed to source the true weights of the holdings.  Fundsmith are under no obligation under UK law to disclose their holdings, but the law in the Unites States of America states that all companies that hold shares in USA companies must declare those holdings, and the amount they own, on a quarterly basis in a '13F' filing.  I found a site called [Whale Wisdom](https://whalewisdom.com/) that shows these filings.

With these found, I performed some cross-validated regression using 80% of the data as training data and a k-fold of 5 (all CV r2 scores came above 98%). The model was tested on the remaining 20% of the data with an r2 score of 99%.  These high scores shouldn't be too surprising; after all, the Fund price *does* move in line with the holding price movements in a linear fashion.

I extracted the coefficients and compared them to the USA holding weights with a bar chart illustrating the results below.

![Model coefficients and actual USA holding weight](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/coefficients_USA_weights_copy.jpg)

You can see the model coefficients are close to their true values for 4 holdings : Becton Dickinson, Intuit, Pepsi and Brown Forman.  The rest range from being 'in the ball park' (Paypal), 'vastly over-estimated' (Stryker, Microsoft) and 'completely wrong' (ADP).  The r2 score between the model coefficients and true weights is 1.8%.

I wondered how much of an affect reducing the number of predictors would have on the model coefficients.  So I created a dictionary and sytematically filled it with keys of the number of predictors, and the values being the r2 score between the model coefficients and the true weights.  Only the holding with the lowest coefficient was removed from the model in each iteration.  A bar chart of the result is below.

![Number of predictors vs R2 score of coefficients and true weights USA](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/r2_scores_num_holdings.png)

The r2 score reduces dramatically when the number of holdings is reduced past 32.  Note that I removed from the plot the r2 values when the number of predictors fell below 12 because they were so negative that they swamped out the rest of the data when plotted.  The maximum r2 score occurs with 39 predictors and has a value of 13.2%. If you are interested in viewing the bar chart of the coefficients and actual weights, open the toggle below.

----
<details>
 <summary><b>Coefficients and true weights with maximum R2 score</b></summary>
 <p>
 <img src=https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/final_coef_weights.png alt="Model Coefficients & true weights for 39 predictors">
 </p>
</details>

----

## 5. Using Facebook's Prophet model to show that 'past performance does not predict future returns'

Facebook released a seasonal modelling library in 2017 called Prophet.  It is intuitive to use and I decided to see what predictions it would make on the Fundsmith Equity Fund data.

I set training data to be from Fund inception in November 2010 to February 2020, right before the drop in the stock market caused by the uncertainty around Coronavirus.  Test data is from February 2020 to December 2020.

If you wish to see the function I created to run the model, please open to toggle below.

----
<details>
 <summary><b>Code of function made with Prophet to forecast the Fundsmith Equity Fund</b></summary>
 <p>
 <img src=https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/proph_code.png alt="Function created with Prophet to predict Fundsmith Equity Fund prices">
 </p>
</details>

----

The training data (black) with forecast (blue) are:

![Training data with Prophet forecast](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/proph_test_preds.png)

And the training data (black), forecast (blue) and test data (red)

![All data with Prophet forecast](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/proph_alldata_preds.png)

As you can see, there is no way the Prophet model (or, for that matter, I believe any model) could predict the affect the Coronavirus had on the market.  What is interesting (and almost certainly coincidental) is that the fund price recovered to around the predicted values.

Prophet also has the opportunity to output some information on the training data.

![All data with Prophet forecast](https://github.com/alexstedman/PersonalProjects/blob/main/Fundsmith_Equity_Project/images/proph_outputs.png)

The first output is the prediction without the training data overlaid.  Nothing much to see here that we don't know from what we've already seen.
The second output is the general trend of the fund price over days of the week.  From this we can immediately discount Saturday and Sunday because trading (and the fund price) do not change then. The fund price tends to make the smallest gains on Thursdays, and the largest gains on Tuesdays.
The third output shows that from the beginning of the year, the fund gains generally increase up to mid-July, and then the gains decrease up to the end of the year.

## Conclusion

Thanks for reading this far.  In conclusion, the linear regression failed to align the coefficients with the actual weights of the stocks in the fund.  Taking the 39 stocks that aligned most with the predicted coefficients resulted in an r2 score of 13%.
Facebook's Prophet model could not predict the drop in the market caused by Coronavirus, but it did manage to predict the price once it had recovered.  This almost entirley coincidental.
