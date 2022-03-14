<img src = 'https://user-images.githubusercontent.com/69991618/111783988-c2483380-88b2-11eb-806b-6e2bbd495298.png' width = 100 alt = 'GA'>

# Indeed Salary scrape and classification project

## Context

This is an assignment that was given to me while studying at General Assembly. The project was split into the following sub-tasks:

1. Build a web scrape and scrape indeed.com

  - Information scraped:

    - Title
    - Location
    - Company
    - Salary

2. Clean and normalise scraped data
3. Fit two different classifier models on cleaned data using only the location as a predictor
4. Create more variables from scraped data to act as predictors
5. Re-fit two different classifier models on data using new predictors
6. Tune re-fitted models to optimise parameters
7. Evaluate re-fitted models

-----

## Web Scrape

Indeed is known for being difficult to scrape, and it is important to rotate through IP addresses to mask your identity otherwise they will give you a captcha. A captcha is a way of preventing bots from scraping their site because it presents the algorithm with a task that is challenging for computers to solve but solvable by humans (although they’re getting harder!). While it is possible to write a programme to solve the Captcha, it’s easier to just avoid the captcha altogether.

To avoid the captcha, I encrypted my connection using ExpressVPN, a paid VPN service provider. I also downloaded tor using the terminal command `brew install tor` and then ran it in my terminal during the scrape. This means my IP address is routed through several different tor nodes and re-assigned at each one, masking my computer’s IP address from Indeed. (Got to avoid that ban!) By connecting to the VPN first, it masks my traffic and identity both from my ISP and potentially malicious tor nodes, adding further security and privacy. Finally, I also updated the user-agent field I sent to indeed to that which my computer usually sends when I am accessing a website with my browser. If I don’t do this Indeed can see that it is a script running and can throw up a captcha.

I scraped through 21 cities: 16 in the UK, 5 in the US and 4 in the EU; and 3 job titles per city: data scientist, data analyst and data engineer. I reset my IP address for every city and if a captcha was thrown. In the case of captcha recurring after successive IP resets, the web scrape broke so I could manually investigate (this never happened). Finally, after scraping through a page, I delayed the scrape from accessing the next page for a random amount of time up to 20 seconds.

Finally, I deduplicated rows and dropped all rows without salary data, leaving a total of 1986 usable results for modelling. I saved this to a csv: `jobs.csv`.

-----

## Data Cleaning

I loaded the data from jobs.csv and did some clean up on the job_title column. This mostly involved using regex to pull job_titles from sentences detailing job descriptions, as well as some minor tweaks to formatting.

I also cleaned the location column to group the jobs into the nearest city of the cities I scraped. 

For the salary column, I dropped all rows where the salary was not quoted in years. Then I analysed each string, and where a range was presented I converted this to the mean between the two values. Otherwise, I just converted the string to a number. Since there were multiple currencies pulled: EUR, GBP and USD, I converted all salaries to USD using the `forex_python.converter` library.

Finally, because we were creating a classifier model I analysed the salary target variable to identify which classes to define as target variables. I observed that the range of salaries was very high, so instead of two classes, I created four predictor classes:
|Category|Salary Band|Percentile|
|----|----|----|
|LOW|<= 48937.11|25th percentile|
|MID-LOW|48937.11 - 63046.50|50th percentile|
|MID-HIGH|63046.50 - 82400.00|75th percentile|
|HIGH|82400.00 < salary|100th percentile|

I classified each row in the `salary_band` column.

I also calculated the baseline accuracy for the dataset. The baseline accuracy of the model is what accuracy we can expect if we always predict the largest group. We can use the baseline accuracy to identify if we are performing better than just guessing the largest category each time. The baseline accuracy for this dataset is 26.03% if you always predict the `MID-LOW` category.

-----

## Location Only

When predicting only the location, I used two models: Logistic Regression and Decision Tree Classifier.

### Logistic Regression

After fitting this model, we received a mean cross-validation score of 0.4446. What this means is that of all the predictions the model made, it was right 44.46% of the time. This may not sound spectacular, but if you guess the largest group each time, you can only expect to be right 26.03% of the time. Therefore being correct 44.46% of the time is a large improvement on the baseline accuracy.

The top 5 most important coefficients for predicting each category are:

|HIGH|MID-HIGH|MID-LOW|LOW|
|-----|-----|-----|-----|
|New York|New Orleans|Paris|Liverpool|
|Chicago|New York|Swansea|Leeds|
|London|Chicago|Liverpool|Glasgow|
|Los Angeles|Glasgow|Glasgow|Belfast|
|Austin|Amsterdam|Leeds|Swansea|

### Decision Tree Classifier

For this model, we had a mean cross-validation score of 0.4484, translating to a model that accurately predicts the correct class 44.84% of the time. Again, against the baseline accuracy, this is a vast improvement and it is even a small improvement on the Logistic Regression model above.

The top 5 most important features for this model are:
New York
London
Austin
Chicago
Los Angeles

I have explained above by what criteria these features are ranked.

-----

## Feature Engineering

After fitting the above models for location only, we were tasked with identifying other variables for predicting salary bands.

I created four predictors:
Seniority_senior
Seniority_junior
Seniority_mid_level
Is_engineer
To create my seniority predictors, I compiled a dictionary with keys: senior, junior; which mapped to lists containing words that indicate if a job is senior or junior. To create my is_engineer predictor, I checked if the title had the word engineer in it, if it did the cell was 1, if not the cell became 0.

-----

## Multi-featured models

### Logistic Regression

I fitted a new Logistic Regression model using the location and the new predictors as predictor variables, and salary_band the target once again. The mean cross-validation score for this model was 0.4943, meaning it predicted correctly 49.43% of the time. This was an improvement of approximately 5% on the first Logistic regression model.

The top 5 predictors for this model were:

|HIGH|MID-HIGH|MID-LOW|LOW|
|-----|-----|-----|-----|
|seniority_senior|location_amsterdam|location_swansea|location_liverpool|
|location_new_york|seniority_senior|location_paris|location_leeds|
|is_engineer|location_new_york|location_glasgow|location_kent|
|location_london|location_chicago|location_new_orleans|location_swansea|
|location_chicago|location_new_orleans|location_liverpool|location_glasgow|

After this, I used a Grid Search (a python class) to optimise my Logistic Regression hyperparameters. After tuning, the best model had a mean cross-validation score of 0.4836. This is lower than the model before tuning, but this is because it applies a penalty function to ensure the model doesn’t overfit the training data. (Overfitting is when the model cannot generalise to unseen data). It also has the advantage of zeroing out redundant columns, making it possible to prune the features in the model and make it run faster. This is important when running models at scale.

### Decision Tree Classifier

For the Decision Tree Classifier, we got a mean cross-validation score of 0.4804. Again an improvement on the location only classifier, and it has comparable predictive ability as the Logistic Regression model above.

I tuned the model from the start, I did not fit a model without running it through Grid Search first.

The top 5 most important features according to this model are:
Seniority_senior
Location_new_york
Is_engineer
Location_chicago
Location_paris

-----

## Model Evaluation

### Logistic Regression

The goal of this model is to predict the salary bands of a job given the location and job title. It is important, however, that the model does not predict the salary will be HIGH when it is not. If it does, this will undermine client trust, whereas if the model predicts LOW and the salary is HIGH, the client will be happy. The main metric that measures this is called precision. This can be defined as, for all the times the model predicts a certain class, how many of those predictions are correct. 

The model initially had a `HIGH` precision of 0.6965, meaning that 69.65% of all cases where the model predicted `HIGH` were correct. 

It is possible to tune this and change the threshold of certainty at which the model will predict whether the class is HIGH or not. I changed the model so that it would predict if the class was HIGH if the probability that it was was greater than 0.6. This raised the precision to 0.8221 - 82.21% of all predictions of HIGH are correct. 

There is a trade-off here - another metric called recall measures of all the instances of a job actually having a HIGH salary, how many of those does the model correctly classify as having a HIGH salary. For the model without a threshold, the recall is 0.7035. This means that of all actual instances of a HIGH salary, the model correctly classifies 70.35% of them. After adjusting the threshold, though, the recall becomes 0.3367 - the model correctly classifies just 33.67% of all actual instances of HIGH salaries as HIGH. It is important to find the right balance between recall and precision to maximise consumer confidence in this model.

### Decision Tree Classifier

This model I did not tune with a threshold, but I will still report on the precision and recall scores.

The precision for predicting HIGH for this model is 0.7253 - 72.53% of all predictions of HIGH are correct predictions. The recall for predicting HIGH for this model is 0.6834 - 68.34% of all instances of HIGH are correctly classified as HIGH.

If I had to pick one model, I would use the Decision Tree Classifier, as it is far more interpretable than a Logistic Regression. Especially for multiclass prediction.
