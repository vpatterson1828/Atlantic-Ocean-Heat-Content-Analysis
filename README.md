# Hurricane and Ocean Heat Content Analysis

## Description
This project analyzes trends in Atlantic Ocean Heat Content (OHC) at the 0–700 meter layer from 2005 to 2024, using regression techniques to study long-term patterns and fluctuations over time and, later, incorporating hurricanes and tropical storms. It aims to explore the relationship between ocean heat dynamics, time, and extreme weather events.

## Requirements
The following R packages are required:
- `ggplot2`
- `lubridate`
- `dplyr`
- `mgcv`

You can install them using:

```r
install.packages(c("ggplot2", "lubridate", "dplyr", "mgcv"))
```
## Setting Up the Data
The following code creates the data frame that will be used for this project. The CSV's required will be attached with this file. Ensure they are in your working directory. 

```{r setup, warning=FALSE, message=FALSE}
set.seed(13)
library(ggplot2)
library(lubridate)
library(dplyr)
library(mgcv)

# Call data
data <- read.csv("merged_data_by_year_month.csv")
ocean_heat <- read.csv("ocean_heat_processed.csv")

# Sort ocean_heat data by date to ensure the sequence is correct
ocean_heat <- ocean_heat[order(ocean_heat$date), ]
# Create a sequential time variable
ocean_heat$time <- seq_len(nrow(ocean_heat))
# Ensure "Date" is formatted correctly
ocean_heat$date <- as.Date(ocean_heat$date)

# Create AOHC bins (e.g., low, medium, high)
data <- data %>%
  mutate(AO_bin = cut(AO, breaks = quantile(AO, probs = c(0, 0.33, 0.66, 1)), 
                      labels = c("Low", "Medium", "High"), include.lowest = TRUE))
# Group Years into decades
data$Year_Group <- ifelse(as.numeric(as.character(data$Year)) <= 2009, "2000s",
                                 ifelse(as.numeric(as.character(data$Year)) <= 2019, "2010s", "2020s"))
# Convert Year_Group to factor
data$Year_Group <- factor(data$Year_Group, levels = c("2000s", "2010s", "2020s"))

# Ensure AO_bin is a factor
data$AO_bin <- factor(data$AO_bin, levels = c("Low", "Medium", "High"))

# Ensure TS_H is a factor
data$TS_H <- as.factor(data$TS_H)

```

## Visualize Data
```{r ocean-heat-over-time, echo=TRUE, warning=FALSE}
# Plot AOHC over time
ggplot(ocean_heat, aes(x = date, y = AO, group=1)) +
  geom_line(color = "darkblue", size = 1) +
  labs(
    title = "Ocean Heat Content Over Time",
    x = "Date",
    y = "Ocean Heat Content (AO)"
  ) +
  theme_minimal()
  
# Boxplot showing interaction between AO_bin and Year_Group
ggplot(data, aes(x = Year_Group, y = Max_Wind, fill = AO_bin)) +
  geom_boxplot() +
  labs(
    title = "Maximum Wind Speed by Ocean Heat Content and Year Group",
    x = "Year Group",
    y = "Maximum Wind Speed (mph)",
    fill = "Ocean Heat Content"
  ) +
  theme_minimal()

# Interaction plot with Year as a continuous variable
data$Year <- as.factor(data$Year)
ggplot(data, aes(x = Year, y = Max_Wind, color = AO_bin, linetype = TS_H, group = interaction(AO_bin, TS_H))) +
  geom_line(stat = "summary", fun = "mean", size = 1) +  # Plot mean Max_Wind for each year
  labs(
    title = "Interaction Plot: Maximum Wind Speed by Year, OHC, and Storm Type",
    x = "Year",
    y = "Average Maximum Wind Speed (mph)",
    color = "Ocean Heat Content",
    linetype = "Storm Type"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) 

# Plot data (Year vs Wind Speed)
ggplot(data, aes(x = Year, y = Max_Wind, color = TS_H)) +
  geom_point(size = 3, alpha = 0.7) +
  labs(title = "Year vs. Maximum Wind Speed",
       x = "Year",
       y = "Maximum Wind Speed (mph)",
       color = "Storm Type") +
  theme_minimal()+ 
  theme(axis.text.x = element_text(angle = 45, hjust = 1))

# Plot data (AO vs Wind Speed)
ggplot(data, aes(x = AO, y = Max_Wind, color = TS_H)) +
  geom_point(size = 3, alpha = 0.7) +
  labs(title = "OHC vs. Maximum Wind Speed by Storm Type",
       x = "Atlantic Ocean Heat Content (AO)",
       y = "Maximum Wind Speed (mph)",
       color = "Storm Type") +
  theme_minimal()

# Density of AOHC
ggplot(data, aes(x = AO)) +
  geom_density(fill = "blue", alpha = 0.5) +  
  labs(
    title = "Density Plot of Storm OHC",
    x = " Storm Ocean Heat Content (OHC)",
    y = "Density"
  ) +
  theme_minimal()
```

## AOHC Temporal Trend Splines
## Introduction to Models

This section employs smoothing splines to analyze the temporal dynamics of Ocean Heat Content (OHC), leveraging the functionality of the **`mgcv`** package. Three models are fitted to capture different aspects of the relationship over time:

1. **First Model**: A **Generalized Additive Model (GAM)** is fitted using the `gam` function to examine the long-term trends in OHC over time.
2. **Second Model**: Another GAM is fitted with an **additive structure**, incorporating both temporal and seasonal effects to capture nuanced variability.
3. **Third Model**: A **Generalized Additive Mixed Model (GAMM)** is used, implemented with the `gamm` function, to account for temporal autocorrelation by including an AR(1) correlation structure. The `gamm` function will allow for incorporation of random effects and hierarchical structures in the model, something proposed to be explored with later analysis. For example, one could model seasonality as a random effect, where each group (e.g., year) has its own seasonal pattern that deviates from the overall trend. In this analysis, though, seasonality was treated as a fixed effect.

As a note, the **`gam`** function in the **`mgcv`** package uses Restricted Maximum Likelihood (REML) for estimating smoothing parameters, providing an efficient framework for modeling additive relationships. In contrast, **`gamm`** combines smoothing with mixed-effects modeling via the **`lme`** function from the **`nlme`** package. While `gamm` also uses REML for estimating random effects and correlation structures, it employs an iterative process to optimize smoothing parameters, making it more flexible but computationally slower.

### Model 1 
This model examines long-term trends in AOHC over time.
```{r, echo=TRUE, warning=FALSE}
# Fit a penalized cubic spline to OHC ~ date
spline_model <- gam(AO ~ s(time, bs = "cr", k = 55), data = ocean_heat)

# View the model summary
summary(spline_model)

# Predict values for visualization
ocean_heat$AO_fitted <- predict(spline_model, newdata = ocean_heat)

# Plot the original data and the fitted spline
ggplot(ocean_heat, aes(x = date)) +
  # Original data points
  geom_point(aes(y = AO), color = "black", alpha = 0.6, size = 1.5) +  
  # Fitted spline line
  geom_line(aes(y = AO_fitted), color = "navyblue", size = 1.2) +  
  # Add labels and title
  labs(
    title = "Model 1: Long-Term Trend in Atlantic Ocean Heat Content",
    subtitle = "Fitted penalized cubic smoothing spline capturing long-term trends (2005–2024)",
    x = "Date",
    y = "Atlantic Ocean Heat Content"
  ) +
  # Customize theme for better aesthetics
  theme_minimal(base_size = 14) +
  theme(
    plot.title = element_text(face = "bold", size = 16, hjust = 0.5),
    plot.subtitle = element_text(size = 12, hjust = 0.5, color = "gray40"),
    axis.title = element_text(face = "bold", size = 14),
    axis.text = element_text(size = 12),
    panel.grid.major = element_line(color = "gray80"),
    panel.grid.minor = element_blank()
  ) 

# Diagnostic plots
plot(spline_model, residuals = TRUE, pch = 16)
gam.check(spline_model)  # Check smoothness and residual patterns

# Extract residuals and fitted values
residuals <- residuals(spline_model)     # Residuals from the model
fitted <- fitted(spline_model)           # Fitted values from the model

# Create residuals vs. fitted values plot
# Residuals vs. Fitted Values Plot
ggplot(data.frame(Fitted = fitted, Residuals = residuals), aes(x = Fitted, y = Residuals)) +
  geom_point(color = "black", alpha = 0.6) +
  geom_hline(yintercept = 0, color = "red", linetype = "dashed") +
  labs(
    title = "Residuals vs. Fitted Values",
    x = "Fitted Values",
    y = "Residuals"
  ) +
  theme_minimal()

# QQ plot of residuals
ggplot(data.frame(Residuals = residuals), aes(sample = Residuals)) +
  stat_qq() +
  stat_qq_line(color = "red") +
  labs(
    title = "QQ Plot of Residuals",
    x = "Theoretical Quantiles",
    y = "Sample Quantiles"
  ) +
  theme_minimal()
```

### Model 2
This model accounts for short-term variations in AOHC (seasonality) by introducing an additive structure.
```{r, echo=TRUE, warning=FALSE}
# Fit the model with seasonality
ocean_heat$months <- as.numeric(format(ocean_heat$date, "%m"))
spline_model_with_seasonality <- gam(
  AO ~ s(time, bs = "cr", k = 45) + s(months, bs = "cc", k = 12), # Add seasonality
  data = ocean_heat
)

# View the model summary
summary(spline_model_with_seasonality)

# Predict values for visualization
ocean_heat$AO_fitted_season <- predict(spline_model_with_seasonality, newdata = ocean_heat)

# Plot the original data and the fitted spline
ggplot(ocean_heat, aes(x = date)) +
  # Original data points
  geom_point(aes(y = AO), color = "black", alpha = 0.6, size = 1.5) +
  # Fitted smoothing spline line
  geom_line(aes(y = AO_fitted_season), color = "navyblue", size = 1.2) +
  # Labels for the plot
  labs(
    title = "Model 2: Long-Term and Seasonal Trends in Ocean Heat Content",
    subtitle = "Penalized cubic smoothing spline for long-term trend and cyclic spline for seasonality",
    x = "Date",
    y = "Atlantic Ocean Heat Content"
  ) +
  # Improved theme for aesthetics
  theme_minimal(base_size = 14) +
  theme(
    plot.title = element_text(face = "bold", size = 16, hjust = 0.5),
    plot.subtitle = element_text(size = 12, hjust = 0.5, color = "gray40"),
    axis.title = element_text(face = "bold", size = 14),
    axis.text = element_text(size = 12),
    panel.grid.major = element_line(color = "gray80"),
    panel.grid.minor = element_blank()
  ) 


# Diagnostic plots
plot(spline_model_with_seasonality, residuals = TRUE, pch = 16)
gam.check(spline_model_with_seasonality)  

# Extract residuals and fitted values
residuals <- residuals(spline_model)     
fitted <- fitted(spline_model)           

# Create residuals vs. fitted values plot
# Residuals vs. Fitted Values Plot
ggplot(data.frame(Fitted = fitted, Residuals = residuals), aes(x = Fitted, y = Residuals)) +
  geom_point(color = "blue", alpha = 0.6) +
  geom_hline(yintercept = 0, color = "red", linetype = "dashed") +
  labs(
    title = "Residuals vs. Fitted Values",
    x = "Fitted Values",
    y = "Residuals"
  ) +
  theme_minimal()

# QQ plot of residuals
ggplot(data.frame(Residuals = residuals), aes(sample = Residuals)) +
  stat_qq() +
  stat_qq_line(color = "red") +
  labs(
    title = "QQ Plot of Residuals",
    x = "Theoretical Quantiles",
    y = "Sample Quantiles"
  ) +
  theme_minimal()

```

### Model 3
This model adds an autoregressive AR(1) structure to effectively model temporal autocorrelation in the residuals.
```{r, echo=TRUE, warning=FALSE}
# AR 1
spline_model_AR<- gamm(
  AO ~ s(time, bs = "cr", k = 45) + s(months, bs = "cc", k = 8),
  correlation = corAR1(form = ~ time),
  data = ocean_heat
)

# Predict fitted values from the GAM component
ocean_heat$AO_fitted_AR <- predict(spline_model_AR$gam, newdata = ocean_heat)

# Plot original data and fitted spline
ggplot(ocean_heat, aes(x = date)) +
  # Original data points in black
  geom_point(aes(y = AO), color = "black", alpha = 0.6, size = 1.5) +
  # Fitted GAM with AR(1) in navy blue
  geom_line(aes(y = AO_fitted_AR), color = "navyblue", size = 1.2) +
  # Add labels and title
  labs(
    title = "Model 3: Long-Term and Seasonal Trends with AR(1) Adjustment",
    subtitle = "Additive smoothing spline for time and seasonality with AR(1) for temporal autocorrelation",
    x = "Date",
    y = "Atlantic Ocean Heat Content "
  ) +
  # Improved theme for aesthetics
  theme_minimal(base_size = 14) +
  theme(
    plot.title = element_text(face = "bold", size = 16, hjust = 0.5),
    plot.subtitle = element_text(size = 12, hjust = 0.5, color = "gray40"),
    axis.title = element_text(face = "bold", size = 14),
    axis.text = element_text(size = 12),
    panel.grid.major = element_line(color = "gray80"),
    panel.grid.minor = element_blank()
  ) 

# Diagnostics for AR1
# Extract fitted values
fitted_vals_AR <- fitted(spline_model_AR$gam)

#Extract residuals
residuals_AR <- residuals(spline_model_AR$lme, type = "normalized")

# Residuals vs. Fitted Values
ggplot(data.frame(Fitted = fitted_vals_AR, Residuals = residuals_AR), aes(x = Fitted, y = Residuals)) +
  geom_point(color = "black", alpha = 0.6, size = 1.5) +  # Black points for residuals
  geom_hline(yintercept = 0, color = "navyblue", linetype = "dashed", size = 1) +  # Navy blue dashed line at 0
  labs(
    title = "Residuals vs. Fitted Values",
    x = "Fitted Values",
    y = "Residuals"
  ) +
  theme_minimal(base_size = 14) +
  theme(
    plot.title = element_text(face = "bold", size = 16, hjust = 0.5),
    axis.title = element_text(face = "bold", size = 14),
    axis.text = element_text(size = 12),
    panel.grid.major = element_line(color = "gray80"),
    panel.grid.minor = element_blank()
  )

# QQ Plot
ggplot(data.frame(Sample = residuals_AR), aes(sample = Sample)) +
  stat_qq(color = "black", size = 1.5, alpha = 0.6) +  # Black points for QQ plot
  stat_qq_line(color = "navyblue", size = 1) +  # Navy blue QQ line
  labs(
    title = "QQ Plot of Residuals",
    x = "Theoretical Quantiles",
    y = "Sample Quantiles"
  ) +
  theme_minimal(base_size = 14) +
  theme(
    plot.title = element_text(face = "bold", size = 16, hjust = 0.5),
    axis.title = element_text(face = "bold", size = 14),
    axis.text = element_text(size = 12),
    panel.grid.major = element_line(color = "gray80"),
    panel.grid.minor = element_blank()
  )

summary(spline_model_AR$gam)

# Set up a 1-row, 2-column layout for the plots
par(mfrow = c(1, 2), mar = c(5, 5, 4, 2) + 0.1) 

# Plot smooth term for time (long-term trend) over the original data
plot(spline_model_AR$gam, select = 1, residuals = FALSE, rug = TRUE,
     pch = 16, cex = 0.7,
     main = "Long-Term Trend in AOHC",
     xlab = "Date (Index)", ylab = "Smooth Term for Time")

# Plot smooth term for months (seasonality) over the original data
plot(spline_model_AR$gam, select = 2, residuals = FALSE, rug = TRUE,
     pch = 16, cex = 0.7,
     main = "Seasonal Variation in AOHC",
     xlab = "Months", ylab = "Smooth Term for Months")

```

Below showcases the autocorrelation plots for all three models.

```{r, echo=TRUE, warning=FALSE}
# ACF plot of residuals (with and without seasonality)
acf(residuals(spline_model), main = "ACF of Residuals for Model 1")
acf(residuals(spline_model_with_seasonality), main = "ACF of Residuals for Model 2")

# Check residual autocorrelation for AR1
acf(residuals(spline_model_AR$lme, type = "normalized"), main = "ACF of Residuals for Model 3")
```

## Storm Splines
The fourth model examines the relationship between AOHC corresponding to hurricane or tropical storm activity and time. 

```{r, echo=TRUE, warning=FALSE}
# Check and convert variables to categorical
data$Year <- as.factor(data$Year)       # Ensure time is categorical
data$TS_H <- as.factor(data$TS_H)  # Ensure storm type is categorical
```

### Model 4
A Gamma Generalized Linear Model (GLM) with a log link function was employed to analyze the data. The `glm` function is a base function in R. It supports a variety of families (e.g., Gaussian, Binomial, Gamma, Poisson) and link functions (e.g., log, identity, logit) to fit generalized linear models. As a note, `glm` utilizes Maximum Likelihood (ML) estimation.
```{r, echo=TRUE, warning=FALSE}

# Fit the Gamma GLM
glm_model <- glm(
  AO ~ Year + Max_Wind + TS_H, 
  data = data,
  family = Gamma(link="log")
)
summary(glm_model)

# Extract residuals from the model
residuals_glm <- residuals(glm_model)
plot(residuals_glm)

# Plot Model
# Add fitted values to the dataset
data$AO_fitted <- fitted(glm_model)
# Observed vs. Predicted Plot
ggplot(data, aes(x = AO_fitted, y = AO)) +
  geom_point(color = "blue", alpha = 0.6) +
  geom_abline(slope = 1, intercept = 0, color = "red", linetype = "dashed") +
  labs(
    title = "Observed vs Predicted Storm OHC",
    x = "Predicted Storm OHC",
    y = "Observed Storm OHC"
  ) +
  theme_minimal()

# Generate a grid of values for Year
year_grid <- data.frame(
  Year = sort(unique(data$Year)),
  Max_Wind = mean(data$Max_Wind),  # Average Max_Wind
  TS_H = "HU"                     # One level of TS_H
)

year_grid$Year <- factor(year_grid$Year, levels = unique(data$Year))

# Predict AO for the grid
year_grid$AO_predicted <- predict(glm_model, newdata = year_grid, type = "response")


ggplot(year_grid, aes(x = Year, y = AO_predicted)) +
  geom_point(color = "darkblue", size = 4, shape = 16) +  
  geom_line(aes(group = 1), color = "darkblue", size = 1.2, linetype = "solid") +  
  labs(
    title = "Effect of Year on AOHC",
    x = "Year",
    y = "Predicted AOHC"
  ) +
  theme_minimal(base_size = 14) +  
  theme(
    plot.title = element_text(face = "bold", size = 16, hjust = 0.5),  
    axis.title = element_text(face = "bold", size = 14),  
    axis.text = element_text(size = 12),  
    axis.text.x = element_text(angle = 45, hjust = 1),  
    panel.grid.major = element_line(color = "gray80"),  
    panel.grid.minor = element_blank()  
  )
```
