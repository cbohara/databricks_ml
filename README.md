# Scalable ML with Apache Spark
## Intro
Send code to driver   
Driver optimizes and distributes code to workers   
Each worker runs a JVM process   

Spark under the hood optimizations   
- Unresolved logical plan = syntax checks
- Logical plan > catalyst optimizer > optimized logic plan   
- Physical plans for cost based optimization   
- Turns into code to be executed   

Delta lake   
Built on top of parquet   
Time travel = helpful to go back to a previous model version   

Machine learning   
- Predict based on patterns 
- Without explicit programming
- Funtion that maps features to outputs   

Supervised learning - Use labeled data mapping input to output - focus on the course   
Unsupervised learning - Find patterns/groupings of unlabeled data   
Semi-supervised learning - Combine labeled and unlabeled data   
Reinforcement learning - Learn from feedback   

ML workflow:   
Define biz use case > prep - define success, identify contraints > collect data > build features out of data > create models > choose the best model > deploy model > evaluate if this model is driving biz value or needs to be retrained    

Want true positives + true negatives   
Any model that has no errors is over fitted and will not generalize well    

## Exploring data
Summary stats:   
mean/average = central tendency for the data point   

stddev - measures how much data points are spread out
- low stddev = data points tend to reside close to the mean = not much spread
- high stddev = data points are spread out over a wide range of values   

interquartile range (IQR) - another measure of variability that is less influenced by outliers       
- arrange the data is ascending order 
- find the median value = split the data set in half   
- then take the median of the first half and second half to generate quarters   
- the median of the first half is first quartile Q1 and the median of the second half is third quartile Q3   
- IQR = Q3 - Q1   
when using summary()
- 25% - 25% of the data falls below this value   
- 50% - 50% of the data falls below this value 
- 75% - 75% of the data falls below this value    

use `dbutils.data.summarize(df)` to get very helpful stats   
for numeric features   
- count 
- missing 
- mean
- std dev 
- zeros
- min 
- median 
- max 

for categorical features 
- count 
- missing 
- unique 

handling null values   
- drop records 
- numeric - replace mean/median/mode/zero 
- categorical - replace with mode = most frequent value in the data set or create a special column for null   
- if you do any imputation techniques, you must include an additional field specifying that the field was imputed   

```
from pyspark.ml.feature import Imputer

# create data frame with missing values 
df = spark.createDataFrame([
    (1.0, float("nan")),
    (2.0, float("nan")),
    (float("nan"), 3.0),
    (4.0, 4.0),
    (5.0, 5.0)
], ["a", "b"])

# specify input columns and new columns to be generated with imputation
# default is to use mean for imputation
imputer = Imputer(inputCols=["a", "b"], outputCols=["out_a", "out_b"])

# fit will compute the mean value for each column
model = imputer.fit(df)

# transform will replace the null value with the mean = default approach 
model.transform(df).show()

print(imputer.explainParams())
```

Spark ML API:   
transformers 
- transforms 1 DF to another 
- accepts DF as input and returns DF with 1+ columns appended 
- simply apply rule based transformations
estimators 
-  an algorithm which can be fit on a dataframe to produce a transformer 
- ex: a learning algo is an estimator which trains on a DF and produces a model   

Use randomSplit to divide the data   
Providing the seed value is arbitrary    
Providing it is optional   
Useful to provide when you need reproducible results   
```
train_df, test_df = airbnb_df.randomSplit([.8, .2], seed=42)
```

Transforming these prices into their logarithmic scale can often help normalize or reduce the skewness, making the distribution more symmetric and easier to work with for analysis purposes.   

By taking the logarithm of price values, you essentially create a new scale that compresses larger values and expands smaller values. This transformation can make the distribution of prices more symmetrical, which can aid in statistical analysis, modeling, or visualizations, especially when dealing with highly skewed or heavily right-tailed distributions.    

```
train_df = train_df.select(log("price").alias("log_price"))
```

Create 1 model that predicts average price and one the predicts median price   
Use RMSE (root mean square error) to evaluate   
Lower RMSE values indicate better model performance, as they signify smaller differences between predicted and observed values    

## Linear Regression
Goal - find the line of best fit   
y = mx + b   
We provide the data - x,y of the data points   
x is the feature and y is the label   
The model finds the m and b - m is the slope and b is the intercept to the y axis   

Goal - minimize the residuals = distance between the best fit line and the data points   
Regression evaluators - measures the closeness between the actual value and predicted error   

Evaluation metrics  
- loss = actual - prediction   
- absolute loss = abs(actual - prediction)
- squared loss = (actual - prediction)^2 - gets rid of the negative and positive errors canceling each other out   
- root mean squared error (RMSE) - fancier version of loss above - varies based on the scale of the data   
- r squared - compare sum of all errors to the average and compare sum of all errors to the prediction - range 0 (no value) - 1 (perfect fit)   

Assumptions
- assume there is a linear relationship 
- observations are independent of each other - means the data points aren't derived from other data points  
- data is normally distributed 
- variance is consistent from one data point to the next   

scikit learn - popular single-node ML library   
ML with spark - scale out + speed up   
- MLlib - based on RDDs - maintenance mode
- Spark ML - use dataframes - active development   

[Spark ML linear regression](https://spark.apache.org/docs/latest/ml-classification-regression.html#linear-regression)   

```
# specify the configuration
lr = LinearRegression(featuresCol="bedrooms", labelCol="price")
# teach the model based on the training data frame the relationship between bedrooms and price
lr_model = lr.fit(train_df)
```

price(y) = coefficient(m) * bedrooms(x) + y-intercept(b)
What does the lr_model contain? 
- coefficients(m) - the value you multiple the number of bedrooms by - ex: if the model learned that each bedroom adds $1000 to the price, then the coefficient would be 1000   
- intercept(b) - constant term - it represented the base price of the house with zero bedrooms   

[VectorAssembler](https://spark.apache.org/docs/latest/ml-features.html#vectorassembler)
Combines a list of columns to create a single column as a list   

ex: hour, mobile, userFeatures becomes features column   

 id | hour | mobile | userFeatures     | clicked | features
----|------|--------|------------------|---------|-----------------------------
 0  | 18   | 1.0    | [0.0, 10.0, 0.5] | 1.0     | [18.0, 1.0, 0.0, 10.0, 0.5]

When working with multiple columns, x is not limited to one column   
It is a vector of all the columns   
price(y) = coefficient(m) * x + y-intercept(b)   

non-numeric features
- categorical - strings with no ordering 
    - can use one hot encoding to create a new column per each value and then set the value to 1 or 0   
    - works when we only have a few values (ie: Animal - dog, cat, bird)
    - use SparseVector when working with large variety of values (ie: all animals in a zoo)
- ordinal features - strings with order (ex: small, medium, large)

convert non-numeric values to numbers   

[StringIdexer](https://spark.apache.org/docs/latest/ml-features.html#stringindexer)    
encodes a string column of labels to a column of label indices   
Ordering options supported
- freqDesc - most frequent label gets 0 - default  
- freqAsc - least frequent label gets 0

Assume that we have the following DataFrame with columns id and category:

```
 id | category
----|----------
 0  | a
 1  | b
 2  | c
 3  | a
 4  | a
 5  | c
```

category with a string column with 3 labels - a, b, c   
a -> 0 - gets 0 because it is the most frequent value   
c -> 1    
b -> 2 - gets 2 because it is the least frequent value   

[OneHotEncoder](https://spark.apache.org/docs/latest/ml-features.html#onehotencoder)
one hot encoder doesn't accept string input, it requires numeric input
so the string indexer is first ran to transform strings into numbers, and then we can do one hot encoding on the number representation of the string

[Pipelines](https://spark.apache.org/docs/latest/ml-pipeline.html)   
Combine multiple algorithms into a single workflow    

Estimator - used to create a transformer 
- ex: LinearRegression is an estimator which trains on a DF and produces a model 
```
lr = LinearRegression(labelCol="price", featuresCol="features")
```

Transformer - algorithm that transforms one DF to another
- ex: ML model is a transformer which when fit on the training DF will create another DF with predictions 
```
lr_model = lr.fit(train_df)
```

Pipeline - chains multiple estimators and transformers together to specify a ML workflow   

https://spark.apache.org/docs/latest/ml-pipeline.html#example-estimator-transformer-and-param
```
from pyspark.ml.linalg import Vectors
from pyspark.ml.classification import LogisticRegression

# Prepare training data from a list of (label, features) tuples.
training = spark.createDataFrame([
    (1.0, Vectors.dense([0.0, 1.1, 0.1])),
    (0.0, Vectors.dense([2.0, 1.0, -1.0])),
    (0.0, Vectors.dense([2.0, 1.3, 1.0])),
    (1.0, Vectors.dense([0.0, 1.2, -0.5]))], ["label", "features"])

# Create a LogisticRegression instance. This instance is an Estimator.
lr = LogisticRegression(maxIter=10, regParam=0.01)

# Print out the parameters, documentation, and any default values.
print("LogisticRegression parameters:\n" + lr.explainParams() + "\n")

# Learn a LogisticRegression model. This uses the parameters stored in lr.
model1 = lr.fit(training)

# Since model1 is a Model (i.e., a transformer produced by an Estimator), we can view the parameters it used during fit().
print("Model 1 was fit using parameters: ")
print(model1.extractParamMap())

# We may alternatively specify parameters using a Python dictionary as a paramMap
paramMap = {lr.maxIter: 20}
# Specify 1 Param, overwriting the original maxIter.
paramMap[lr.maxIter] = 30  
# Specify multiple Params.
paramMap.update({lr.regParam: 0.1, lr.threshold: 0.55})  

# Prepare test data to validate the model
test = spark.createDataFrame([
    (1.0, Vectors.dense([-1.0, 1.5, 1.3])),
    (0.0, Vectors.dense([3.0, 2.0, -0.1])),
    (1.0, Vectors.dense([0.0, 2.2, -1.5]))], ["label", "features"])

# Make predictions on test data using the Transformer.transform() method.
prediction = model2.transform(test)
result = prediction.select("features", "label", "prediction") \
    .collect()
for row in result:
    print("features=%s, label=%s -> prob=%s, prediction=%s"
          % (row.features, row.label, row.myProbability, row.prediction))
```

Steps 
1. Define algorithm used to estimate/predict the labelCol based on the input of the featuresCol- `lr = LinearRegression(labelCol="price", featuresCol="features")`
2. Create a model which essentially is just a dataframe with a column added with the predictions - `lr_model = lr.fit(train_df)`

## ML Flow 
Open source   
Created by Databricks folks   

Core ML learning challenges 
- Keeping track of experiments/model development
- Reproducing the code + data used 
- Compare one model to another 
- Standardize packaging + deploying the model

Components 
- Tracking - record experiments 
- Projects - package model
- Model - the model itself  
- Model registry - model lifecycle management 

Track ML development with 1 line of code   
```
mlflow.autolog()
```

Captures
- who ran the model on what data source  
- what parameters were used 
- how did it perform?
- what was the env used when running the model?   

`Reproduce Run` button makes it easy to reproduce the run   

Makes it easy to identify the best performing model   

Still need to do data prep + featurization - leverage time travel in delta lake   
Still need to use a framework to create the model - Spark ML, skikitlarn, tensorflow, etc    
Use ML flow more for tracking model creation + deploying it to production   

Model artifact content   
Includes trained ML model along with its metadata   
SparkML stores data as snappy parquet file by default   
Metadata is stored as JSON file   

input_example.json shows the column names and first 5 rows of the data used to train the model   

MLmodel  file contains the system details    
```artifact_path: model
databricks_runtime: 12.2.x-cpu-ml-scala2.12
flavors:
  python_function:
    data: sparkml
    env:
      conda: conda.yaml
      virtualenv: python_env.yaml
    loader_module: mlflow.spark
    python_version: 3.9.5
  spark:
    code: null
    model_data: sparkml
    pyspark_version: 3.3.2.dev0
mlflow_version: 2.1.1
model_uuid: 88b66ba84b8242ba8cdfc6f123f9428f
run_id: 51ea9527aeaf48aeb887065bf80b2adf
saved_input_example_info:
  artifact_path: input_example.json
  pandas_orient: split
  type: dataframe
utc_time_created: '2024-02-09 15:53:59.427769'
```

A model artifact typically includes the data trained on and metadata (which model used, parameters used, etc) but not the original source code used to train the model    
This is sufficient to make transform the DF to make predictions    
A serialized model contains all the infor required to recreate the model's behavior   

MLflow model registry   
- Centralized place to store models
- Facilitates collaboration and observable experimentation
- Able to move from testing to production 
- Can have approval/governance workflows 
- Model ML deployments and their performance in the wild

Can see what model is used in production and iterative models that are in dev/testing phase   

Promoting to production can be done manually via UI or programmatically via CI/CD process   

MLFlow tracking used to track model development  
Once a model is ready to be used, then MLFlow model registry is used to manage the model lifecycle from dev > preprod > prod    

## [AutoML](https://docs.databricks.com/en/machine-learning/automl/index.html)
Quickly verify the predictive power of a data set   
Get a baseline model to guide project direction   

Throw data into AutoML to get V0 model   
Then use the notebook to refine the model   

Provide data set + prediction target   
AutoML takes care of 
- feature engineering
- training, tuning, and evaluating multiple models 
- displays the results + provides Python notebook with the source code for each trial run 
- calcs summary stats  

Types of models 
- Classification - predict a category
- Regression models - predict a number
- Forecasting models - predict future values based on historical data

Samples large data sets because works on a single node   

Outputs 
- Generates DataExploration notebook 
- Shares evidence of multiple models and their performance   
- Provides best performing notebook
