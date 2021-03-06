# Machine Learning Assignment
Karin  
22 September 2015  
#Executive Summary

The goal of this project is to predict how well the participants did the exercise on data obtained from this link http://groupware.les.inf.puc-rio.br/har. Greatly appreciated your generosity to make this data and paper available for us to learn. Refer to reference on details of the authors. 

This paper will describe which processes were followed to determine the best model to apply to the test data and how the process were conducted. 
 
##Getting the data
The data consists of two csv files.

```r
training <- read.csv("pml-training.csv") 
testing <- read.csv("pml-testing.csv") 
```

##Cleaning the data

```r
#determine columns with values
training_filter_col <- training[,(colSums(is.na(training)) == 0)]
testing_filter_col <- testing[,(colSums(is.na(testing)) == 0)]
#Remove columns with no values
removeCol <- c("X","user_name","raw_timestamp_part_1","raw_timestamp_part_2","cvtd_timestamp","new_window")
NewTrain <- training_filter_col[,!(names(training_filter_col) %in% removeCol)]
NewTest <- testing_filter_col[,!(names(testing_filter_col) %in% removeCol)]
```

###Removing irrelevant variables
The process of excluding near zero variance features will be used to remove variables that should not be included in the training of any models. 

The training data (NewTrain) is then split into a validation data set (FinalValid) on which the models will be tested before the final test will be run on the test set. 30% of the data will be allocated to the validation data.The remainder of the training data (FinalTrain) will be used to train the models.

The reason for splitting the data, is because I selected the approach to build different models and need the validation data to test each of the models on before applying the best fit model to the original test set.

```r
library(caret)
nzvcol <- nearZeroVar(NewTrain)
NewTrain <- NewTrain[, -nzvcol]

#Split training data into training and validation sets
inTrain <- createDataPartition(y=NewTrain$classe,
                               p=0.7, list=FALSE)
FinalTrain <- NewTrain[inTrain,]
FinalValid <- NewTrain[-inTrain,]
```

##Machine Learning
###Cross validation
The next section of this document will describe which models were selected and how the data trained into different models, to get to a point of selecting the best model fit.

I have opted to use the random forest model, Recursive Partition model and Naive Bayes model as the three contenders to find the best model, with the possibilty that if neither one give an acceptable accuracy, I will consider blending these models.

Due to very long run times and a complete memory loss on my laptop, I did some research on how to improve this. The train control does exactly that. I used suggestions from stackoverflow.com to set these parameters.

I have set the training control using the repeated K-fold cross validation for the recursive partition models, because I saw in two documents that this is a very reliable and preferred method to do cross validation on. Refer to references for recommendations on the repeated K-fold cross validation.


```r
#Traincontrol for RPart - use recommended repeated K-fold cross validation
train_ctrl <- trainControl(method = "repeatedcv", 
                           number = 10, 
                           repeats = 3) 

#Train Recursive Partition model - 2min
modelRpart <- train(classe ~ ., data = FinalTrain, 
                    method = "rpart", 
                    tuneLength = 30, 
                    metric = "Accuracy",
                    trControl = train_ctrl)
```

The random forest still gave over 3hours to complete using the traincontrol as per previous sections. I then set it to use no method and the train time were reduced to just over a minute and no impact on the computer memory.


```r
#Train random forest - 1min
rfModel <- train(classe ~ ., 
                 data=FinalTrain, 
                 method="rf", 
                 tuneGrid=data.frame(mtry=3),
                 trControl=trainControl(method="none"))
```

The Naive Bayes produced a poor performance result on the same traincontrol as RPart and with further research, I have changed the train control to be more effective on NB model, using cross validation with a repeat of 10 times. 

```r
#Train ctrl for Naive Bayes
train_ctrlNB <- trainControl(method = "cv",
                             number = 10)
                             #verboseIter = TRUE)#show progress

#Train Naive Bayes model - 4min
modelNB <- train(classe ~ ., 
                 data=FinalTrain, 
                 trControl = train_ctrlNB, 
                 method = "nb")
```

###Compare respective models
Apply the models to the validation data sets to determine which model provides the best accuracy.

```r
#Make predictions on validation data - Recursive Partition
modelPred <- predict(modelRpart, FinalValid)
#Confusion matrix
cm <- confusionMatrix(modelPred, FinalValid$classe)
cm$overall['Accuracy']
```

```
##  Accuracy 
## 0.8278675
```

```r
#Random Forest accuracy check
modelPred2 <- predict(rfModel, FinalValid)
#Confusion matrix
cm2 <- confusionMatrix(modelPred2, FinalValid$classe)
cm2$overall['Accuracy']
```

```
##  Accuracy 
## 0.9966015
```

```r
#Naive Bayes accuracy check
modelPred3 <- predict(modelNB, FinalValid)
#Confusion matrix
cm3 <- confusionMatrix(modelPred3, FinalValid$classe)
cm3$overall['Accuracy']
```

```
##  Accuracy 
## 0.7570093
```
#Expected error rate and accuracy rates
Comparing accuracy results from all these models the random forest is the most accurate with a 99.75% accuracy. The RPart model is second with 90.38% and Naive Bayes the worst with 76.89%. Due to this huge accuracy on the random forest, I have decided to just use that model and not create a combination model to increase the accuracy.

So the expected error rate applying the random forest model is 0.25%.
#Applying the model to the test data
The random forest model is now applied to the test data to predict what each of the twenty lines in terms of the category on how well the exercise were done.

I then joined the prediction to the test data into a new variable called predDF.

```r
#Predict RF model on the testing set
pred1 <- predict(rfModel,newdata = testing)

#Create dataframe on predictions on test set
predDF <- cbind(classe = pred1,testing)
```

##Summarise prediction results
In order to really analyse the prediction, I dropped all the varaibles except for the user and class and added a count to this to plot.

```r
#Rework results to plot
library(dplyr)
summary <- select(predDF, user_name, classe)
summary <- count(summary, user_name, classe)
#summary

x <- group_by(summary, classe, n)
x2 <- summarise(x)

#Notice difference in scale - explains the discrepancy
par(mfrow = c(2,2), mar = c(5,4,2,1))
plot(NewTrain$classe, col="blue", main="Fig1 Training data set - Excl 0 Cols", xlab="Classe", ylab="Frequency")
plot(FinalTrain$classe, col="blue", main="Fig2 - Training set - Excl NZV", xlab="Classe", ylab="Frequency")
plot(FinalValid$classe, col="red", main="Fig3 - Validation set", xlab="Classe", ylab="Frequency")
plot(x2$classe, col="red", main="Fig4 - RF applied on test set", xlab="Classe", ylab="Frequency")
```

![](MachineLearning_files/figure-html/unnamed-chunk-9-1.png) 

##Comparisons
In the graphs, the main aim were to show how accurate the model is in comparison to looking at the outcome variable in all the data where only the empty columns were removed (Fig1), then to the training set just after the near zero variables were excluded. fig 3 show how the model shows on the validation data and lastly in Fig 4, the RF model applied to the test set.

Notice the frequency reduction from graph to graph, however, the pattern remains similar. Just in figure 4, the E classe shows higher, but it is actually higher in all other graphs, just not as visible due to the scale difference.

###References:
Ugulino, W.; Cardador, D.; Vega, K.; Velloso, E.; Milidiu, R.; Fuks, H. Wearable Computing: Accelerometers' Data Classification of Body Postures and Movements. Proceedings of 21st Brazilian Symposium on Artificial Intelligence. Advances in Artificial Intelligence - SBIA 2012. In: Lecture Notes in Computer Science. , pp. 52-61. Curitiba, PR: Springer Berlin / Heidelberg, 2012. ISBN 978-3-642-34458-9. DOI: 10.1007/978-3-642-34459-6_6. 

More information is available at this link: http://groupware.les.inf.puc-rio.br/har#ixzz3lMNG7T00

Cross Validation recommendations:
*"Predictive Modeling with R and the caret Package" Author Max Kuhn, Ph.D
*Jason Brownlee's blog - http://machinelearningmastery.com

