# Dailly Stock Index Trend Prediction
#### - Hanna Lee, Nhi Nguyen, Vichitra Kumar, and Ruirui Zhang - 
## Introduction
Using resources available on Reddit, a social news discussion website, and Sequenced Packet Exchange (SPX), a networking protocol, we performed analysis on the potential predictive variables of daily stock index trend. The process and results have been discussed and demonstrated in the below report.
## Problem/Motivation Statement
The use of innovative resources like Reddit and Sequenced Packet Exchange (SPX) for analyzing potential predictive variables of daily stock index trends has greatly inspired our team. This report's findings demonstrate the value of leveraging technology to enhance analytical capabilities and drive insights that can have a significant impact on financial decision-making. We are motivated to continue exploring and utilizing cutting-edge machine learning tools to drive more comprehensive and insightful analysis, and ultimately help traders and investors make more informed investment decisions.

## Dataset and Analytic Goals
#### Dataset
Our dataset is a combination of two separate datasets:
- SPX dataset: daily stock index data from 2017-01-01 to 2022-12-31. This dataset was obtained through the Bloomberg terminal in the form of a csv file.
- Reddit dataset: Reddit dataset was scraped using specific keywords for date range 2017-01-01 to 2022-12-31 by the Pushshift API, which is a Reddit data archive service. API provides the functionality to scrape submission and comments separately with a date range. Below is the structure of a datapoint in this dataset:
      
      {
           'created_utc': date,
           'title_list': [],
           'self_text': [],
           'comments': []
           'spx_index': index
      }
#### Analytic Goals
- Are interest rate hike/cut related words useful in predicting whether stock index increases or decreases the next day?
- Are encouraging buy words and discourage sell words useful in predicting whether stock index increases or decreases the next day? Is there any correlation between the word counts and the index change?
- Utilizing Reddit users sentiment to predict price action.
- Judging from Reddit users' usage of swing trade fu words like “pump n dump” “pull out”, gauging how much price action fluctuates within a short time frame.

## Overview of data engineering pipeline
1. Scrape the Reddit data
a. define key words list for filtering
b. scrape for comment, posts and titles separately
2. Obtain the SPX index data from Bloomberg terminal, the data range from 2017-01-01 to 2022-12-31.
3. Define a function that reads the json file and csv file and pushes it to the GCP bucket.
4. Retrieve data from GCP bucket, pre-process and convert it into aggregations.
5. Insert the aggregations into a database hosted on MongoDB Atlas.
6. Put all the above functions together and wrap them in a DAG function.
7. Trigger the DAG in Airflow.

## Preprocessing goals, algorithms, and time efficiency
#### Preprocessing goal: get a bag-of-words random forest baseline. Clean the data frame:
 a) Convert the datetime data to date type.
 
 b) Build prediction target value: window function over date col and get the difference between the next day and today, fill the nan value (nan value is because of the holidays and weekends). Fill the nan value with backward fill. 1 represents the index goes up or remains the same, 0 represents the index goes down.

 c) Merge the comments, title, posts together: concat three columns and clean all the irrelevant punctuation and links, the new column is a string.

 d) Tokenize texts and clean text: Tokenize and remove the stop words, stem the words.

 e) Vectorize text data: Use the basic CountVectorizer to vectorize the text data.
#### Initial Algorithm
Build RF model:

 a) Build train/test dataset: Take the data from 2017-01-01 to 2021-12-31as the train dataset, the data from 2022-01-01 to 2022-06-30 as the test dataset.
 
 b) Calculate the IDF of the text vector: use the IDF estimator to fit on the train data and use the transformer to transform test data.
 
 c) Build a random forest with parameters tuned by random grid search: Customize a Random Grid class to do random grid search. Set the search range of maxDepth from 10 to 30, the range of numTrees from 10 to 100. Do the cross validation to tune hyperparameters.
 
 d) Evaluation: the auc-roc score of the baseline model is 0.445. Since the data label is imbalanced, this is a score like a random guess.
 
Time efficiency: 3.27 hours on CPU cluster, 2 workers.

## Machine Learning Goals, Outcomes, and Execution Time Efficiency
### 1. Building a time series model to predict SPX index with Reddit users sentiment
#### Sentiment Analysis
Goal:
- Build a time series based model to predict SPX index using reddit user’s sentiment from
the text.

Data Processing Steps:

1. Text aggregation: Concatenate the reddit submission and comments text at the daily level for analysis
2. Pre-processing: Pre-processed the data to remove blanks, spaces, stop words, symbols, links etc.
3. Text tokenization - tokenized the text using BertTokenizer API
4. Sentiment Analysis from finance perspective: Utilized transfer learning from BERT model
i.e. finbert to get daily sentiments. finbert is a transformer model specifically trained on financial data to predict the sentiment of financial text. It provides sentiments as three labels ie. Positive, Negative and Neutral.

Time Efficiency: To predict the sentiment, it takes 15 min to run on a GPU with 3 workers.
#### Multivariate Time-Series Model
Trained a multivariate time series model to predict the SPX index based on the user sentiments.
- Data Processing: created a Prophet model object and added the regressor to the model.
- Model Parameters: As per time series data, model parameters were added like daily
seasonality, holidays, changepoint range tec
- Fit the Prophet model on our data with 3 sentiment labels, date and y variable(SPX price).
- Prediction: Predict the next 15 days SPX price to evaluate our model.
- Performance: Model has mean absolute error on the lower side i.e. 46.2%. It can be
increased using cross validation to fine tune our model.
#### Conclusion:
User sentiment plays an important role in predicting the performance of market and stocks but sentiment alone is not enough to predict the close SPX index. If it can be merged with other market indicators, it can be useful in stock market prediction or SPX index prediction.

### 2. Are encouraging buy words and discouraging sell words useful in predicting whether stock index increases or decreases the next day?
#### Goals
- Find whether the out-vote number of buy to sell is correlated to the up-and-downs of the day.
- Build a prediction model using only the word count of encouraging words and discouraging words.
#### Data preparation
- Get the words count column: define a udf to get the words counts. New column is in the format of dictionary
- Sum the number of keywords appearances: define another udf to sum the counts of the defined keywords lists.
- Subtract the discouraging words count from the encouraging words counts.
#### Analysis
- Calculate the correlation between the index difference and the (encouraging word count - discouraging word count), the correlation is 0.02. It's a very small positive correlation.
- Build a random forest model and get the auc_roc score. The auc_roc score is 0.50.
#### Conclusion
- There is a slight positive correlation between the out-vote number and the index change.
- Performance of the random forest model with only these two word counts surpasses the baseline model by 0.06, meaning these two features are useful for index change prediction.
#### Time efficiency:
The random forest model spent 6.8 minutes on fitting without parameter tuning on the CPU cluster, 2 workers.

### 3. Building a logistic regression model to predict the stock index trend
- We wanted to find out if the words in our “rate hike” and “rate cut”’ were useful in predicting whether the stock index went up or down the next day.
- After cleaning the combined text and storing words in a list for each day, we created two new features, which were the numbers of times the words in our “rate hike” and “rate cut” lists appeared on Reddit that day. We called these new features rate_hike_word_count and rate_cut_word_count.
- We assembled these two features into a vector and performed min-max scaling, then split the data into training (80%) and validation (20%) sets.
- We fit a logistic regression model to the training data using the following parameters:
      - Regularization parameter: 0.01
      - Maximum iteration: 1000
      - Threshold: 0.55
      - Fit intercept: True
- We then used the trained model to predict the validation data set, which resulted in an area under the ROC curve of 0.51. This was higher than the result obtained from fitting our initial random forest model. It was still not a good result; however, for this type of data which was heavily driven by human behaviors, we did not expect high performance metrics for our first try.
- We concluded that words that are related to interest rate hike and rate cut could be useful in predicting the next day's stock index trend. This was not surprising since traders’ decisions to buy, hold, or sell could be affected by the information they were exposed to on Reddit.
- Our logistic regression model took 1.63 seconds to train on 1,439 training data points and 1.38 minutes to predict 382 validation data points on CPU cluster, 2 workers.

### 4. Analysis and Visualization of Magnitude Change in Stock Prices with Respect to the Number of Times “Swing Trade” words is Used.
(*Swing trading: when traders trade according to the momentum and take profit within a short timeframe.)
#### Goal
Using the number of times the “Swing Trade” words are seen in Reddit, we try to see if there is a correlation where the more time these words appear in Reddit, the fluctuation in the stock price within the next 24 hours has a bigger magnitude change.
#### Data preparation
- Form a word count column “sum_swing_trade_terms” that sums up the number of occurrences of the words in the checklist.
- Sum the number of keywords appearances: define a udf to sum the counts of the defined keywords lists.
- Form a price difference column by calculating the price difference between the date and its next day.
#### Analysis and Visualization:
We segmentized the graph into 4 parts, where the first part shows relatively smaller fluctuation in price difference(orange line) as the “Swing Trade” words are used less. Going towards the end of 2018, more occurrences of the “Swing Trade” words are seen, during which we can observe from the graph that the fluctuation of the prices are also relatively larger. Lastly, we see a dip in the word usage and the fluctuation in price difference went down and then both went up as well.

![alt text](https://github.com/nhipqnguyen/Dailly_Stock_Index_Trend_Prediction/edit/main/images/swing_trade_terms.png)
