---
layout:     post
title:      "[R]时间序列数据的读取"
subtitle:   " \"Data analysis!\""
date:       2015-12-07
author:     "fibears"
header-img: "img/in-post/header/r-time-series.jpg"
tags:
    - 数据分析
    - Article
    
---

我在之前的[博文](http://blog.revolutionanalytics.com/2015/08/plotting-time-series-in-r.html)中介绍了如何利用`dygraphs`包绘制时间序列数据，但是在该篇文章我并没有介绍如何利用R读取金融数据。在此文的评论区中，Achim Zeileis 指出R语言内置了许多处理时间序列数据的包，他认为每个人都应该了解它们。因此，本篇博文的主要目的在于详细介绍这些内置的软件包。首先，我将利用上篇博文中所提到的 IBM 和 Linkedin 的股票数据。雅虎财经网站上提供了各个公司的历史股票数据，因此我们只需要通过url获取网站上的`csv`文件。

Achim 还提到：
	如果你已经拥有相关网站的链接，那么你可以利用`zoo`包读取整合这些股票数据，然后再利用`ggplot2`绘制图像。

```r
# Get IBM and Linkedin stock data from Yahoo Finance
ibm_url <- "http://real-chart.finance.yahoo.com/table.csv?s=IBM&a=07&b=24&c=2010&d=07&e=24&f=2015&g=d&ignore=.csv"
lnkd_url <- "http://real-chart.finance.yahoo.com/table.csv?s=LNKD&a=07&b=24&c=2010&d=07&e=24&f=2015&g=d&ignore=.csv"
# 加载zoo
library("zoo")
z <- merge(
	ibm = read.zoo(ibm_url, header = TRUE, sep = ",")$Close,
	lnkd = read.zoo(lnkd_url, header = TRUE, sep = ",")$Close
)
library("ggplot2")
# 时间趋势图的结果如下所示。
autoplot(z, facets = NULL)
```

上述代码中，我们只用了三行代码就完成了数据获取和预处理的过程，然后再利用一句简单的命令就可以得到一张默认的趋势图。

当然，手动获取链接的方法是没有效率的，我们可以利用`tseries`包中`get.historic.quote()`函数自动获取目标数据。

```r
library(tseries)
ibm2 <- get.hist.quote("IBM", quote="Close", provider="yahoo", start="2010-08-24", end="2015-09-02",retclass="zoo")
```

如果你希望获得和股票交易软件一样的界面，那么你可以利用`quantmod`包来实现。

```r
library(quantmod)
IBM <- getSymbols("IBM",env=NULL)
# getSymbols可以获取“xts/zoo”时间序列对象
head(IBM)
```

此外，`IBM`还是一个 “OHLC” 对象，我们可以利用`chartSeries()`函数绘制标准的股票趋势图。

```r
is.OHLC(IBM)
chartSeries(IBM, type="candle", subset="2010-08-24::2015-09-02")
```

函数`getSymbols()`不仅可以从 Yahoo, Google, FRED 和 oanda 等金融服务网站中抓取数据，还可以读取 MySQL, .csv, 和 RData 等类型的数据。

[Quandl](https://www.quandl.com)是获取免费金融数据的最佳网站。只要你登陆该网站，就可以利用下面的代码抓取相应的金融数据。

```r
library(Quandl)
token <- "your_token_string"
# Authenticate your token
Quandl.auth(token) 
ibmQ <- Quandl("WIKI/IBM", start_data = "2010-08-24", end_date = "2015-09-03")
head(ibmQ)

```

本文中，我只介绍了几个获取金融数据的基础方法，在`R`语言中还有许多抓取和分析金融时间序列数据的工具。

最后需要注意的是：几年前我曾写过一个关于如何使用`Quandl R API`的简明教程。但不幸的是，由于`Quandl`架构发生了变动，所以我的代码已经无法运行。对其感兴趣的朋友可以参考以下代码，该代码主要用来绘制亚洲国家货币汇率的趋势图。

```r

#token <- 'Token_From_Quandl'			# Sign up with Quandl to get a token
token <- "my_token_string"
#-----------------------------------------------------
library(Quandl)					# Quandl package
library(ggplot2)				# Package for plotting
library(reshape2)				# Package for reshaping data
#
Quandl.auth(token)				# Authenticate your token

# Construct Quandl code for first currency
currency <- c("BK73","JYD","HDD","BK64","BK66","NDD","SGD","TWD","BK72")
codes <- paste("BOE/","XUDL",currency,sep="")

# Function to fetch major currency rates from Quandl
readQC <- function(curr,start="2010-08-31",end="2015-08-31"){
  N_curr <- length(curr)
  d <- Quandl(codes[1],start_date=start,end_date=end )[,1]
  A <- array(0,dim=c(length(d),N_curr))
  for(i in 1:N_curr){
      # Get the rates
      A[,i] <- Quandl(curr[i],start_date=start,end_date=end)[,2]
    }
  df <- data.frame(d,A)
  return(df)
}
rates <- readQC(codes)			# Fetch the currency rates

# Prepare data for plotting
names(rates) <-c("Date",currency)
rates$Date <- as.Date(rates$Date)		# Make Date into type Date
meltdf <- melt(rates,id="Date")		# Shape data for plottting
#
ggplot(meltdf,aes(x=Date,y=value)) + 
  geom_line(aes(colour=variable)) +
  ylab("USD")+
  scale_colour_brewer(palette="Set1") +
  ggtitle("Daily Exchange Rates for Various Asian Currencies") 

```

原文链接：http://blog.revolutionanalytics.com/2015/09/reading-financial-time-series-with-r.html

原文作者：Joseph Rickert

翻译：Fibears




