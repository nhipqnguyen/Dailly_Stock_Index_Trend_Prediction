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

Build a time series based model to predict SPX index using reddit user’s sentiment from
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
1. Data Processing: created a Prophet model object and added the regressor to the model.
2. Model Parameters: As per time series data, model parameters were added like daily
seasonality, holidays, changepoint range tec
3. Fit the Prophet model on our data with 3 sentiment labels, date and y variable(SPX price).
4. Prediction: Predict the next 15 days SPX price to evaluate our model.
5. Performance: Model has mean absolute error on the lower side i.e. 46.2%. It can be
increased using cross validation to fine tune our model.
#### Conclusion:
User sentiment plays an important role in predicting the performance of market and stocks but sentiment alone is not enough to predict the close SPX index. If it can be merged with other market indicators, it can be useful in stock market prediction or SPX index prediction.
