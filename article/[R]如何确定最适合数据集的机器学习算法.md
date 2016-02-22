# [R]如何确定最适合数据集的机器学习算法

原文链接：http://machinelearningmastery.com/spot-check-machine-learning-algorithms-in-r/

原文作者：Jason Brownlee

译者：Fibears

抽查(Spot checking)机器学习算法是指如何找出最适合于给定数据集的算法模型。

本文中我将介绍八个常用于抽查的机器学习算法，文中还包括各个算法的 R 语言代码，你可以将其保存并运用到下一个机器学习项目中。

## 适用于你的数据集的最佳算法
你无法在建模前就知道哪个算法最适用于你的数据集。

你必须通过反复试验的方法来寻找出可以解决你的问题的最佳算法，我称这个过程为 spot checking。

我们所遇到的问题不是`我应该采用哪个算法来处理我的数据集？`，而是`我应该抽查哪些算法来处理我的数据集？`

### 抽查哪些算法？
首先，你可以思考哪些算法可能适用于你的数据集。

其次，我建议尽可能地尝试混合算法并观察哪个方法最适用于你的数据集。

- 尝试混合算法(如事件模型和树模型)
- 尝试混合不同的学习算法(如处理相同类型数据的不同算法)
- 尝试混合不同类型的模型(如线性和非线性函数或者参数和非参数模型)

让我们具体看下如何实现这几个想法。下一章中我们将看到如何在 R 语言中实现相应的机器学习算法。

## 如何在 R 语言中抽查算法？
R 语言中存在数百种可用的机器学习算法。

如果你的项目要求较高的预测精度且你有充足的时间，我建议你可以在实践过程中尽可能多地探索不同的算法。

通常情况下，我们没有太多的时间用于测试，因此我们需要了解一些常用且重要的算法。

本章中你将会接触到一些 R 语言中经常用于抽查处理的线性和非线性算法，但是其中并不包括类似于boosting和bagging的集成算法。

每个算法都会从两个视角进行呈现：

1. 常规的训练和预测方法
2. `caret`包的用法

你需要知道给定算法对应的软件包和函数，同时你还需了解如何利用`caret`包实现这些常用的算法，从而你可以利用`caret`包的预处理、算法评估和参数调优的能力高效地评估算法的精度。

本文中将用到两个标准的数据集：

- 回归模型：BHD(Boston Housing Dataset)
- 分类模型: PIDD(Pima Indians Diabetes Dataset)

本文中的算法将被分成两组进行介绍：

- 线性算法：简单、较大的偏倚、运算速度快
- 非线性算法：复杂、较大的方差、高精确度

下文中的所有代码都是完整的，因此你可以将其保存下来并运用到下个机器学习项目中。

### 线性算法
这类方法对模型的函数形式有严格的假设条件，虽然这些方法的运算速度快，但是其结果偏倚较大。

这类模型的最终结果通常易于解读，因此如果线性模型的结果足够精确，那么你没有必要采用较为复杂的非线性模型。

#### 线性回归模型
`stat`包中的`lm()`函数可以利用最小二乘估计拟合线性回归模型。

```
# load the library
library(mlbench)
# load data
data(BostonHousing)
# fit model
fit <- lm(mdev~>, BostonHousing)
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, BostonHousing)
# summarize accuracy
mse <- mean((BostonHousing$medv - predictions)^2)
print(mse)

# caret
# load libraries
library(caret)
library(mlbench)
# load dataset
data(BostonHousing)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.lm <- train(medv~., data=BostonHousing, method="lm", metric="RMSE", preProc=c("center", "scale"), trControl=control)
# summarize fit
print(fit.lm)
```

#### 罗吉斯回归模型
`stat`包中`glm()`函数可以用于拟合广义线性模型。它可以用于拟合处理二元分类问题的罗吉斯回归模型。

```
# load the library
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# fit model
fit <- glm(diabetes~., data=PimaIndiansDiabetes, family=binomial(link='logit'))
# summarize the fit
print(fit)
# make predictions
probabilities <- predict(fit, PimaIndiansDiabetes[,1:8], type='response')
predictions <- ifelse(probabilities > 0.5,'pos','neg')
# summarize accuracy
table(predictions, PimaIndiansDiabetes$diabetes)

# caret
# load libraries
library(caret)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.glm <- train(diabetes~., data=PimaIndiansDiabetes, method="glm", metric="Accuracy", preProc=c("center", "scale"), trControl=control)
# summarize fit
print(fit.glm)
```

#### 线性判别分析
`MASS`包中的`lda()`函数可以用于拟合线性判别分析模型。

```
# load the libraries
library(MASS)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# fit model
fit <- lda(diabetes~., data=PimaIndiansDiabetes)
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, PimaIndiansDiabetes[,1:8])$class
# summarize accuracy
table(predictions, PimaIndiansDiabetes$diabetes)

# caret
# load libraries
library(caret)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.lda <- train(diabetes~., data=PimaIndiansDiabetes, method="lda", metric="Accuracy", preProc=c("center", "scale"), trControl=control)
# summarize fit
print(fit.lda)
```

#### 正则化回归
`glmnet`包中的`glmnet()`函数可以用于拟合正则化分类或回归模型。

分类模型：

```
# load the library
library(glmnet)
library(mlbench)
# load data
data(PimaIndiansDiabetes)
x <- as.matrix(PimaIndiansDiabetes[,1:8])
y <- as.matrix(PimaIndiansDiabetes[,9])
# fit model
fit <- glmnet(x, y, family="binomial", alpha=0.5, lambda=0.001)
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, x, type="class")
# summarize accuracy
table(predictions, PimaIndiansDiabetes$diabetes)

# caret
# load libraries
library(caret)
library(mlbench)
library(glmnet)
# Load the dataset
data(PimaIndiansDiabetes)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.glmnet <- train(diabetes~., data=PimaIndiansDiabetes, method="glmnet", metric="Accuracy", preProc=c("center", "scale"), trControl=control)
# summarize fit
print(fit.glmnet)
```

回归模型：

```
# load the libraries
library(glmnet)
library(mlbench)
# load data
data(BostonHousing)
BostonHousing$chas <- as.numeric(as.character(BostonHousing$chas))
x <- as.matrix(BostonHousing[,1:13])
y <- as.matrix(BostonHousing[,14])
# fit model
fit <- glmnet(x, y, family="gaussian", alpha=0.5, lambda=0.001)
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, x, type="link")
# summarize accuracy
mse <- mean((y - predictions)^2)
print(mse)

# caret
# load libraries
library(caret)
library(mlbench)
library(glmnet)
# Load the dataset
data(BostonHousing)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.glmnet <- train(medv~., data=BostonHousing, method="glmnet", metric="RMSE", preProc=c("center", "scale"), trControl=control)
# summarize fit
print(fit.glmnet)
```

### 非线性算法
非线性算法对模型函数形式的限定较少，这类模型通常具有高精度和方差大的特点。

#### k近邻法
`caret`包中的`knn3()`函数并没有建立模型，而是直接对训练集数据作出预测。它既可以用于分类模型也可以用于回归模型。

分类模型：

```
# knn direct classification

# load the libraries
library(caret)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# fit model
fit <- knn3(diabetes~., data=PimaIndiansDiabetes, k=3)
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, PimaIndiansDiabetes[,1:8], type="class")
# summarize accuracy
table(predictions, PimaIndiansDiabetes$diabetes)

# caret
# load libraries
library(caret)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.knn <- train(diabetes~., data=PimaIndiansDiabetes, method="knn", metric="Accuracy", preProc=c("center", "scale"), trControl=control)
# summarize fit
print(fit.knn)
```

回归模型：

```
# load the libraries
library(caret)
library(mlbench)
# load data
data(BostonHousing)
BostonHousing$chas <- as.numeric(as.character(BostonHousing$chas))
x <- as.matrix(BostonHousing[,1:13])
y <- as.matrix(BostonHousing[,14])
# fit model
fit <- knnreg(x, y, k=3)
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, x)
# summarize accuracy
mse <- mean((BostonHousing$medv - predictions)^2)
print(mse)

# caret
# load libraries
library(caret)
data(BostonHousing)
# Load the dataset
data(BostonHousing)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.knn <- train(medv~., data=BostonHousing, method="knn", metric="RMSE", preProc=c("center", "scale"), trControl=control)
# summarize fit
print(fit.knn)
```

#### 朴素贝叶斯算法
`e1071`包中的`naiveBayes()`函数可用于拟合分类问题中的朴素贝叶斯模型。

```
# load the libraries
library(e1071)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# fit model
fit <- naiveBayes(diabetes~., data=PimaIndiansDiabetes)
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, PimaIndiansDiabetes[,1:8])
# summarize accuracy
table(predictions, PimaIndiansDiabetes$diabetes)

# caret
# load libraries
library(caret)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.nb <- train(diabetes~., data=PimaIndiansDiabetes, method="nb", metric="Accuracy", trControl=control)
# summarize fit
print(fit.nb)
```

#### 支持向量机算法
`kernlab`包中的`ksvm()`函数可用于拟合分类和回归问题中的支持向量机模型。

分类模型：

```
# Classification Example:
# load the libraries
library(kernlab)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# fit model
fit <- ksvm(diabetes~., data=PimaIndiansDiabetes, kernel="rbfdot")
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, PimaIndiansDiabetes[,1:8], type="response")
# summarize accuracy
table(predictions, PimaIndiansDiabetes$diabetes)

# caret
# load libraries
library(caret)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.svmRadial <- train(diabetes~., data=PimaIndiansDiabetes, method="svmRadial", metric="Accuracy", trControl=control)
# summarize fit
print(fit.svmRadial)
```
回归模型：

```
# Regression Example:
# load the libraries
library(kernlab)
library(mlbench)
# load data
data(BostonHousing)
# fit model
fit <- ksvm(medv~., BostonHousing, kernel="rbfdot")
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, BostonHousing)
# summarize accuracy
mse <- mean((BostonHousing$medv - predictions)^2)
print(mse)

# caret
# load libraries
library(caret)
library(mlbench)
# Load the dataset
data(BostonHousing)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.svmRadial <- train(medv~., data=BostonHousing, method="svmRadial", metric="RMSE", trControl=control)
# summarize fit
print(fit.svmRadial)
```

#### 分类和回归树
`rpart`包中的`rpart()`函数可用于拟合CART分类树和回归树模型。

分类模型：

```
# load the libraries
library(rpart)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# fit model
fit <- rpart(diabetes~., data=PimaIndiansDiabetes)
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, PimaIndiansDiabetes[,1:8], type="class")
# summarize accuracy
table(predictions, PimaIndiansDiabetes$diabetes)

# caret
# load libraries
library(caret)
library(mlbench)
# Load the dataset
data(PimaIndiansDiabetes)
# train
set.seed(7)
control <- trainControl(method="cv", number=5)
fit.rpart <- train(diabetes~., data=PimaIndiansDiabetes, method="rpart", metric="Accuracy", trControl=control)
# summarize fit
print(fit.rpart)
```

回归模型：

```
# load the libraries
library(rpart)
library(mlbench)
# load data
data(BostonHousing)
# fit model
fit <- rpart(medv~., data=BostonHousing, control=rpart.control(minsplit=5))
# summarize the fit
print(fit)
# make predictions
predictions <- predict(fit, BostonHousing[,1:13])
# summarize accuracy
mse <- mean((BostonHousing$medv - predictions)^2)
print(mse)

# caret
# load libraries
library(caret)
library(mlbench)
# Load the dataset
data(BostonHousing)
# train
set.seed(7)
control <- trainControl(method="cv", number=2)
fit.rpart <- train(medv~., data=BostonHousing, method="rpart", metric="RMSE", trControl=control)
# summarize fit
print(fit.rpart)
```

### 其他算法
R 语言中还提供了许多`caret`可以使用的机器学习算法。我建议你去探索更多的算法，并将其运用到你的下个机器学习项目中。

[Caret Model List](https://topepo.github.io/caret/modelList.html)

这个网页上提供了`caret`中机器学习算法的函数和其相应软件包的映射关系。你可以通过它了解如何利用`caret`构建机器学习模型。

## 总结
本文中介绍了八个常用的机器学习算法：

- 线性回归模型
- 罗吉斯回归模型
- 线性判别分析
- 正则化回归
- k近邻
- 朴素贝叶斯
- 支持向量机
- 分类和回归树

从上文的介绍中，你可以学到如何利用 R 语言中的包和函数实现这些算法。同时你还可以学会如何利用`caret`包实现上文提到的所有机器学习算法。最后，你还可以将这些算法运用到你的机器学习项目中。

## 你的下一步计划？
你有没有试验过本文中的算法代码？

1. 打开你的 R 语言软件。
2. 输入上文中的代码并运行之。
3. 查看帮助文档学习更多的函数用法。


