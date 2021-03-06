##Predicting Barbell Lift Form

###Background
Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement - a group of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their behavior, or because they are tech geeks. One thing that people regularly do is quantify how much of a particular activity they do, but they rarely quantify how well they do it. The goal of this project is to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. More information is available from the website here: http://groupware.les.inf.puc-rio.br/har.

###Exploring and Pre-processing the data
After importing the training data we see that there are 19622 observations of 160 variables.  Exploring the training dataset shows there are bad/messy values in the dataset (eg. '#DIV/0!').  To handle these we reimport the data and set these values to NAs at import.  We suspect the first seven columns of the data will have no predictive value so we remove those columns.  Further exploration shows there are a large number of variables missing data (eg. NAs).  To help with the computation of the models downstream we decide to eliminate any variables where greater than 75% of the observations are NA.  After checking the remaining variables for near zero variance we are left with 53 columns which we will include in our models.  We perform these same preprocessing steps on the testing dataset and also remove the problem_id variable which was unique to the testing set.

```{r,}
library(caret)
# read in raw training data
raw <- read.csv("pml-training.csv", na.strings=c("NA","#DIV/0!",""))
#dimensions of data
dim(raw)
# remove first 7 columns - not relevant to prediction
training <- raw[,8:160]
# convert the outcome variable to a factor
training$classe <- as.factor(training$classe)
# Remove columns with more than 75 % NA values
training2 <- training[,colMeans(is.na(training)) < .75]
# check for near zero variance predictor variables
nearZeroVar(training2, names = TRUE)
raw2 <- read.csv("pml-testing.csv", na.strings=c("NA","#DIV/0!",""))
# remove first 7 variables
testing <- raw2[,8:160]
# set aside problem id field - may need for submission later on
prob_id <- testing$problem_id
# only include columns in testing data that were used in training data
testing2 <- testing[,names(testing) %in% names(training2)]

```
###Partitioning the data
To assess how our models may perform we split the training dataset into two subsets.  The myTrain data contains 60% and the myTest data contains 40% of the training data.

```{r, cache=TRUE}
set.seed(3456)
trainIndex <- createDataPartition(training2$classe, p = .6,
                                  list = FALSE,
                                  times = 1)

myTrain <- training2[ trainIndex,]
myTest  <- training2[-trainIndex,]

```
###Model building
The outcome variable of interest is the classe variable which has values of A,B,C,D, or E.  Our objective is to predict the correct class variable and we decide to use three tree based models for this classification problem.

* An initial CART model
* A tuned CART model using cross validation
* A random forest model using cross validation

#### Initial CART model
We train a CART model using all the default parameters on the myTrain dataset and then predict on the myTest dataset.  This model results in an overall accuracy of .748 and a Kappa value of .6813.

```{r, cache=TRUE}
# try an initial cart model without tuning
set.seed(3456)
rpartFull <- rpart(classe ~ ., data = myTrain)
rpartPred <- predict(rpartFull, myTest, type = "class")
confusionMatrix(rpartPred, myTest$classe)
```
####Tuning parameters
Next we see if we can improve on these results by tuning the model using cross validation.  We set the train control to use 10 fold cross validation with 3 repeats.  We will use these parameters for our remaining models.

```{r, cache=TRUE}
# create train control to perform 3 repeats of 10 fold cross validation
cvCtrl <- trainControl(method = "repeatedcv", repeats = 3,
                       summaryFunction = multiClassSummary,
                       classProbs = TRUE)
```

####Tuned CART model
Running a tuned CART model and predicting on the myTest data results in an improved accuracy of .8306 and a Kappa value of .7859.

```{r, cache=TRUE}
# train CART model using cross validation train control
set.seed(3456)
rpartTune <- train(classe ~ ., data = myTrain, method = "rpart",
                   tuneLength = 30,
                   metric = "Kappa",
                   trControl = cvCtrl)
# run tuned model against test partition
rpartPred2 <- predict(rpartTune, myTest, type = "raw")
confusionMatrix(rpartPred2, myTest$classe)
```

####Random Forest Model
Finally we run a Random Forest model with 10 fold cross validation with 3 repeats. Using the model to predict on the myTest data results in accuracy of .9918 and a Kappa value of .9897.  

```{r,cache=TRUE}
set.seed(3456)
# train Random Forest model using cross validation train control
rfMod = train(classe ~ .,data=myTrain,method="rf", trControl = cvCtrl)
# run model against test partition
rfPred <- predict(rfMod, myTest)
confusionMatrix(rfPred, myTest$classe)
```
###Final Model Selection
To compare models we perform resampling to compare estimated performance of the top two models - the Tuned CART model and the Random Forest Model.

```{r, cache = TRUE}
resamps <- resamples(list(RF = rfMod, TunedCART = rpartTune))
summary(resamps)
```

The performance of the Random Forest model is impressive and based on the resampling we can see that it out performs the tuned CART model.  Given that we split the training data into a roughly 60/40 split and used cross validation to tune our model, we decide to use the Random Forest Model to predict on the testing dataset.  We expect very good (although likely not quite as good) performance using this model on the out of sample test data.

```{r}
# use Random Forest model to predict on testing data set
testPred <- predict(rfMod, newdata = testing2, type = "raw")

```