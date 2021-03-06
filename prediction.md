# Weight Lifting Exercise Data Prediction
Carlos Macasaet  
21 September 2014  

# Summary

This document details the creation of a predictive model that identifies whether
or not a subject is performing a correct Unilateral Dumbbell Biceps Curl and if
not, which of four common mistakes the subject is making. This study is based on
the [Human Activity Recognition](http://groupware.les.inf.puc-rio.br/har)
project.


```r
library( randomForest )
library( dplyr )
```


```r
if( !file.exists( 'data' ) )
{
  dir.create( 'data' )
}
if( !file.exists( 'data/pml-training.csv' ) )
{
  download.file( 'https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv',
                 'data/pml-training.csv', method='curl' )
}
if( !file.exists( 'data/pml-testing.csv' ) )
{
  download.file( 'https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv',
                 'data/pml-testing.csv', method='curl' )
}
```

## Preprocessing

The predefined testing and training sets represent missing values in various
ways. I account for that when reading the files.


```r
train <- read.csv( 'data/pml-training.csv', na.strings=c( '#DIV/0!', 'NA' ) )
test <- read.csv( 'data/pml-testing.csv', na.strings=c( '#DIV/0!', 'NA' ) )
train$user_name <- factor( train$user_name )
test$user_name <- factor( test$user_name )
```

## Modeling

I chose to start modeling with random forest because it is easy to use and
generally performs well out of the box without requiring in-depth domain
knowledge. It had the added benefit of simplifying feature selection. There are
160 features in the raw training set. However, because random forest does not
support missing values, I was able to constrain myself to features for which
all the training and test records had values.

From all the models, I excluded the record identifier (X) and any time-related
features. Although I was open to evaluating time-series models if necessary.

### Baseline Models

#### Random Forest on sensible columns without nulls

The first model I tested was a user-agnostic random forest model. This model
used 55 features and yielded an Out of Bag estimate of error rate of 0.15%. This
is pretty good.

<pre>
Call:
 randomForest(formula = classe ~ ., data = modified_train) 
               Type of random forest: classification
                     Number of trees: 500
No. of variables tried at each split: 7

        OOB estimate of  error rate: 0.15%
Confusion matrix:
     A    B    C    D    E  class.error
A 5579    1    0    0    0 0.0001792115
B    3 3793    1    0    0 0.0010534633
C    0    6 3416    0    0 0.0017533606
D    0    0   13 3202    1 0.0043532338
E    0    0    0    5 3602 0.0013861935
</pre>

#### Random Forest with user-level information

The second model I tested included user information. Depending on the intended
application of this model, this may not be a viable feature. It assumes that
predictions will only be made against users against whom the model has already
been trained. This improved the Out of Bag estimate of error rate to 0.14%.
More interestingly, however, this model was able to predict a correct biceps
curl 100% of the time.

<pre>
Call:
 randomForest(formula = classe ~ ., data = modified_train) 
               Type of random forest: classification
                     Number of trees: 500
No. of variables tried at each split: 7

        OOB estimate of  error rate: 0.14%
Confusion matrix:
     A    B    C    D    E class.error
A 5580    0    0    0    0 0.000000000
B    4 3793    0    0    0 0.001053463
C    0    7 3415    0    0 0.002045587
D    0    0   11 3204    1 0.003731343
E    0    0    0    4 3603 0.001108955
</pre>

### Imputation

The model I chose in the end extended the notion of relying on user information.
I noticed that for many of the columns with missing values had values when the
"new_window" variable was "yes". I decided to test how effective the model would
be if I imputed that feature by creating a simple lookup table of user name and
window number to the value for each of these variables. I used this technique to
fill in missing values on both the training and test sets.

#### Build Lookup Table


```r
new_window_train <- filter( train, new_window == 'yes' )
new_window_candidate_columns <- colSums( is.na( new_window_train ) ) == 0
new_window_candidate_columns[ c( 'X', 'raw_timestamp_part_1',
                                 'raw_timestamp_part_2', 'cvtd_timestamp',
                                 'new_window' ) ] <- FALSE
rownames( new_window_train ) <-
  paste( new_window_train$user_name, new_window_train$num_window, sep='_' )
```

#### Roll Up Missing Values to the Window Level


```r
rollup_windows <- function( data_frame )
{
  function( column_name )
  {
    column <- data_frame[ , column_name ]
    if( sum( is.na( column ) ) == 0 )
    {
      return( column )
    }
    else
    {
      impute <- function( user_name, num_window, original_value )
      {
        if( is.na( original_value ) )
        {
          return( new_window_train[ paste( user_name, num_window, sep='_' ),
                                    column_name ] )
        }
        else
        {
          return( original_value )
        }
      }
      return( mapply( impute, data_frame$user_name, data_frame$num_window,
                      column ) )
    }
  }
}
middle_train <-
  sapply( colnames( train )[ 4:length( train ) - 1 ], rollup_windows( train ) )
imputed_train <-
  cbind( train[ , 1:3 ], middle_train, classe=train$classe )

middle_test <-
  sapply( colnames( test )[ 4:length( test ) - 1 ], rollup_windows( test ) )
imputed_test <-
  cbind( test[ , 1:3 ], middle_test, problem_id=test$problem_id )

candidate_columns <- colSums( is.na( imputed_test ) ) == 0
candidate_columns[ c( 'X', 'raw_timestamp_part_1', 'raw_timestamp_part_2',
                      'cvtd_timestamp', 'new_window' ) ] <- FALSE

modified_train <- imputed_train[ , candidate_columns ]
```


```r
fit <- randomForest( classe ~ ., data=modified_train )
```

### Cross Validation and Out of Sample Error

The model has an Out of Bag estimate of error rate of 0.05%. Because random
forest uses a bootstrapping approach, it is robust against overfitting. The Out
of Bag estimate is an unbiased estimate of the Out of Sample Error. In fact,
when evaluated against the 20 test records, the model correctly predicted the
biceps curl classes for all of them.


```r
fit
```

```
## 
## Call:
##  randomForest(formula = classe ~ ., data = modified_train) 
##                Type of random forest: classification
##                      Number of trees: 500
## No. of variables tried at each split: 7
## 
##         OOB estimate of  error rate: 0.05%
## Confusion matrix:
##      A    B    C    D    E class.error
## A 5580    0    0    0    0   0.0000000
## B    2 3795    0    0    0   0.0005267
## C    0    4 3418    0    0   0.0011689
## D    0    0    2 3213    1   0.0009328
## E    0    0    0    1 3606   0.0002772
```


