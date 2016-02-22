# 如何提升R代码的运算效率？

原文作者：Selva Prabhakaran

原文链接：http://datascienceplus.com/strategies-to-speedup-r-code/

译者：fibears

众所周知，当我们利用R语言处理大型数据集时，for循环语句的运算效率非常低。有许多种方法可以提升你的代码运算效率，但或许你更想了解运算效率能得到多大的提升。本文将介绍几种适用于大数据领域的方法，包括简单的逻辑调整设计、并行处理和Rcpp的运用，利用这些方法你可以轻松地处理1亿行以上的数据集。

让我们尝试提升往数据框中添加一个新变量过程(该过程中包含循环和判断语句)的运算效率。下面的代码输出原始数据框：

```
# Create the data frame
col1 <- runif (12^5, 0, 2)
col2 <- rnorm (12^5, 0, 2)
col3 <- rpois (12^5, 3)
col4 <- rchisq (12^5, 2)
df <- data.frame (col1, col2, col3, col4)
```

逐行判断该数据框(df)的总和是否大于4，如果该条件满足，则对应的新变量数值为'greater_than_4'，否则赋值为'lesser_than_4'。

```
# Original R code: Before vectorization and pre-allocation
system.time({
  for (i in 1:nrow(df)) { # for every row
    if ((df[i, 'col1'] + df[i, 'col2'] + df[i, 'col3'] + df[i, 'col4']) > 4) { # check if > 4
      df[i, 5] <- "greater_than_4" # assign 5th column
    } else {
      df[i, 5] <- "lesser_than_4" # assign 5th column
    }
  }
})
```

本文中所有的计算都在配置了2.6Ghz处理器和8GB内存的MAC OS X中运行。

## 向量化处理和预设数据库结构
循环运算前，记得预先设置好数据结构和输出变量的长度和类型，千万别在循环过程中渐进性地增加数据长度。接下来，我们将探究向量化处理是如何提高处理数据的运算速度。

```
# after vectorization and pre-allocation
output <- character (nrow(df)) # initialize output vector
system.time({
  for (i in 1:nrow(df)) {
    if ((df[i, 'col1'] + df[i, 'col2'] + df[i, 'col3'] + df[i, 'col4']) > 4) {
      output[i] <- "greater_than_4"
    } else {
      output[i] <- "lesser_than_4"
    }
  }
df$output})
```

![](http://i13.tietuku.com/49f787506283ba6a.png)

## 将条件语句的判断条件移至循环外

将条件判断语句移至循环外可以提升代码的运算速度，接下来本文将利用包含100,000行数据至1,000,000行数据的数据集进行测试：

```
# after vectorization and pre-allocation, taking the condition checking outside the loop.
output <- character (nrow(df))
condition <- (df$col1 + df$col2 + df$col3 + df$col4) > 4  # condition check outside the loop
system.time({
  for (i in 1:nrow(df)) {
    if (condition[i]) {
      output[i] <- "greater_than_4"
    } else {
      output[i] <- "lesser_than_4"
    }
  }
  df$output <- output
})
```
![](http://i13.tietuku.com/6c98a6109fc9838c.png)

## 只在条件语句为真时执行循环过程
另一种优化方法是预先将输出变量赋值为条件语句不满足时的取值，然后只在条件语句为真时执行循环过程。此时，运算速度的提升程度取决于条件状态中真值的比例。

本部分的测试将和case(2)部分进行比较，和预想的结果一致，该方法确实提升了运算效率。

```
output <- c(rep("lesser_than_4", nrow(df)))
condition <- (df$col1 + df$col2 + df$col3 + df$col4) > 4
system.time({
    for (i in (1:nrow(df))[condition]) {  # run loop only for true conditions
        if (condition[i]) {
            output[i] <- "greater_than_4"
        } 
    }
    df$output 
})
```

![](http://i13.tietuku.com/612bc92c47ed77d2.png)

## 尽可能地使用 ifelse()语句
利用`ifelse()`语句可以使你的代码更加简便。`ifelse()`的句法格式类似于`if()`函数，但其运算速度却有了巨大的提升。即使是在没有预设数据结构且没有简化条件语句的情况下，其运算效率仍高于上述的两种方法。

```
system.time({
  output <- ifelse ((df$col1 + df$col2 + df$col3 + df$col4) > 4, "greater_than_4", "lesser_than_4")
  df$output <- output
})
```

![](http://i13.tietuku.com/6f210da3858394b8.png)

## 使用 which()语句
利用`which()`语句来筛选数据集，我们可以达到`Rcpp`三分之一的运算速率。

```
# Thanks to Gabe Becker
system.time({
  want = which(rowSums(df) > 4)
  output = rep("less than 4", times = nrow(df))
  output[want] = "greater than 4"
}) 
# nrow = 3 Million rows (approx)
   user  system elapsed 
  0.396   0.074   0.481 
```

## 利用apply族函数来替代for循环语句
本部分将利用`apply()`函数来计算上文所提到的案例，并将其与向量化的循环语句进行对比。该方法的运算效率优于原始方法，但劣于`ifelse()`和将条件语句置于循环外端的方法。该方法非常有用，但是当你面对复杂的情形时，你需要灵活运用该函数。

```
# apply family
system.time({
  myfunc <- function(x) {
    if ((x['col1'] + x['col2'] + x['col3'] + x['col4']) > 4) {
      "greater_than_4"
    } else {
      "lesser_than_4"
    }
  }
  output <- apply(df[, c(1:4)], 1, FUN=myfunc)  # apply 'myfunc' on every row
  df$output <- output
})
```

![](http://i13.tietuku.com/93827a536f6d3c1f.png)

## 利用compiler包中的字节码编译函数cmpfun()
这可能不是说明字节码编译有效性的最好例子，但是对于更复杂的函数而言，字节码编译将会表现地十分优异，因此我们应当了解下该函数。

```
# byte code compilation
library(compiler)
myFuncCmp <- cmpfun(myfunc)
system.time({
  output <- apply(df[, c (1:4)], 1, FUN=myFuncCmp)
})
```

![img6](http://i13.tietuku.com/9535098be8be677d.png)


## 利用Rcpp
截至目前，我们已经测试了好几种提升运算效率的方法，其中最佳的方法是利用`ifelse()`函数。如果我们将数据量增大十倍，运算效率将会变成啥样的呢？接下来我们将利用`Rcpp`来实现该运算过程，并将其与`ifelse()`进行比较。

```
library(Rcpp)
sourceCpp("MyFunc.cpp")
system.time (output <- myFunc(df)) # see Rcpp function below
```
下面是利用`C++`语言编写的函数代码，将其保存为“MyFunc.cpp”并利用`sourceCpp`进行调用。

```
// Source for MyFunc.cpp
#include 
using namespace Rcpp;
// [[Rcpp::export]]
CharacterVector myFunc(DataFrame x) {
  NumericVector col1 = as(x["col1"]);
  NumericVector col2 = as(x["col2"]);
  NumericVector col3 = as(x["col3"]);
  NumericVector col4 = as(x["col4"]);
  int n = col1.size();
  CharacterVector out(n);
  for (int i=0; i 4){
      out[i] = "greater_than_4";
    } else {
      out[i] = "lesser_than_4";
    }
  }
  return out;
}
```

![img7](http://i13.tietuku.com/0831b22958b15999.png)

## 利用并行运算
并行运算的代码：

```
# parallel processing
library(foreach)
library(doSNOW)
cl <- makeCluster(4, type="SOCK") # for 4 cores machine
registerDoSNOW (cl)
condition <- (df$col1 + df$col2 + df$col3 + df$col4) > 4
# parallelization with vectorization
system.time({
  output <- foreach(i = 1:nrow(df), .combine=c) %dopar% {
    if (condition[i]) {
      return("greater_than_4")
    } else {
      return("lesser_than_4")
    }
  }
})

df$output <- output
```

## 尽早地移除变量并恢复内存容量

在进行冗长的循环计算前，尽早地将不需要的变量移除掉。在每次循环迭代运算结束时利用`gc()`函数恢复内存也可以提升运算速率。

## 利用内存较小的数据结构
`data.table()`是一个很好的例子，因为它可以减少数据的内存，这有助于加快运算速率。

```
dt <- data.table(df)  # create the data.table
system.time({
  for (i in 1:nrow (dt)) {
    if ((dt[i, col1] + dt[i, col2] + dt[i, col3] + dt[i, col4]) > 4) {
      dt[i, col5:="greater_than_4"]  # assign the output as 5th column
    } else {
      dt[i, col5:="lesser_than_4"]  # assign the output as 5th column
    }
  }
})
```

![img8](http://i13.tietuku.com/c69924165ed4f939.png)


## 总结
方法：速度， nrow(df)/time_taken = n 行每秒

- 原始方法：1X, 856.2255行每秒(正则化为1)
- 向量化方法：738X, 631578行每秒
- 只考虑真值情况：1002X，857142.9行每秒
- ifelse：1752X，1500000行每秒
- which：8806X，7540364行每秒
- Rcpp：13476X，11538462行每秒



