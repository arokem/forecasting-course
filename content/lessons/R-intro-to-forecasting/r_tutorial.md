---
title: "R Tutorial"
weight: 3
type: book
summary: R tutorial on making forecasts from time-series models using the forecast package
show_date: false
editable: true
---

## Video Tutorials

<iframe width="560" height="315" src="https://www.youtube.com/embed/kyPg3jV4pJ8" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<iframe width="560" height="315" src="https://www.youtube.com/embed/govzki35PIQ" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Text Tutorial

### Setup

```r
library(forecast)
library(ggplot2)

data = read.csv("portal_timeseries.csv", stringsAsFactors = FALSE)
data$date = as.Date(data$date, format = "%m/%d/%Y")
head(data)
NDVI_ts = ts(data$NDVI, start = c(1992, 3), end = c(2014, 11), frequency = 12)
```

### Steps in forecasting

1. Problem definition
2. Gather information
3. Exploratory analysis
4. Choosing and fitting models
5. Make forecasts
6. Evaluate forecasts

### Exploratory analysis

* Process we went through over the last few weeks
* Look at the data structure

```r
plot(NDVI_ts)
acf(NDVI_ts)
```

### Choose and fit models

* Simples model was white noise or the "naive" model:

  > y_t = c + e_t, where e_t ~ N(0, sigma)

* Fit this model using `Arima()`

```r
avg_model = Arima(NDVI_ts, c(0, 0, 0))
str(avg_model)
```


### Make forecasts

* To make forecasts from a model we ask what the model would predict at the time-step
* For the average model we just need to know what c is, which is just the mean(NDVI_ts)

```r
avg_forecast = forecast(avg_model)
str(avg_forecast)
```

* Model object has information on
  * Method used for forecasting
  * Values for fitting the model
  * Information about the model
  * Mean values for the forecast

* The expected value, or point forecast, is in `$mean`

```r
avg_forecast$mean
```

* Change the number of time-steps in the forecast using h

```r
avg_forecast = forecast(avg_model, h = 50)
```

#### Visualize

```r
plot(NDVI_ts)
lines(avg_forecast$mean, col = 'pink')
```


* Better to use built-in plotting functions

```r
plot(avg_forecast)
autoplot(avg_forecast)
```


#### Uncertainty

* Important to know how confident our forecast is
* Shaded areas provide this information
* Only variation in e_t is included, not errors in parameters
* By default 80% and 95%
* Can change using `level`

```r
avg_forecast
avg_forecast <- forecast(avg_model, level = c(50, 95))
avg_forecast
plot(avg_forecast)
```

* Does it look like 95% of the empirical points fall within the gray band?
* How do we tell?
* We'll come back to this when we learn how to evaluate forecasts

### Forecasting with more complex models

* Non-seasonal ARIMA
* `y_t = c + b1 * y_t-1 + b2 * y_t-2 + e_t`

> Instructors note: Actually `y_t = (1 - b1 - b2) * c + b1 * y_t-1 + b2 * y_t-2 + e_t` due to non-zero mean

> Have students build a non-seasonal ARIMA model: 36 month horizon, 80 and 99% prediction intervals
> Then discuss.

#### How this forcast works

```r
arima_model = auto.arima(NDVI_ts, seasonal = FALSE)
arima_model
arima_forecast = forecast(arima_model)
plot(arima_forecast)
```

* Forecast one step into the future
* Use the forecast value as `y_t-1` to forecast second step
* Can see the model at work
* First step influenced strongly postively by previous time-step which is high so above mean
* Second step is pulled below negative AR2 parameter
* Gradually reverts to the mean

#### Seasonal ARIMA

* Best model we found last time
* Not much better than non-seasonal when looking at fit to data

```r
seasonal_arima_model = auto.arima(NDVI_ts)
seasonal_arima_forecast = forecast(seasonal_arima_model, h = 36, level = c(80, 99))
plot(seasonal_arima_forecast)
```
* Do you think it might be a better model for forecasting?
* We'll find out how to tell next week.

### Forecasts from cross-sectional approach

* Just predictor variables, not time-series component
* Predict NDVI based on rain data
* Southern Arizona's vegetation relies on summer monsoons, so focus on that precip

```r
library(dplyr)
library(lubridate)

data$date = as.Date(data$date, "%m/%d/%Y")
monsoon_data <- data %>%
  mutate(month = month(date), year = year(date)) %>% 
  filter(month %in% c(7, 8, 9)) %>%
  group_by(year) %>%
  summarize(monsoon_rain = sum(rain), monsoon_ndvi = mean(NDVI))
```

* Visualize the relationship

```r
ggplot(monsoon_data, aes(x = monsoon_rain, y = monsoon_ndvi)) +
  geom_point() +
  geom_smooth(method = "lm")
```

* Fit a linear model

```r
rain_model = lm('monsoon_ndvi ~ monsoon_rain', data = monsoon_data)
```

* Make a forecast using that model
* Requires forecast values for precipition

```r
rain_forecast = forecast(rain_model, newdata = data.frame(monsoon_rain = c(120, 226, 176, 244)))
plot(rain_forecast)
```
