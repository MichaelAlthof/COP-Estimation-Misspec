
[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **COPretaparch** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml

Name of Quantlet : COPretaparch

Published in : 'Estimation of the Dependence Parameter in Bivariate Archimedean Copula Models under
Misspecification'

Description : 'COPretaparch fits a AR(1)-GJR-GARCH(2,1) model to the daily returns of the indices
Dow Jones (DJ) and DAX and a AR(1)-GJR-GARCH(1,1) model to the daily returns of the stocks of
Volkswagen (VW) and Thyssen-Krupp (TK). The considered time span is 26.08.2005 to 13.08.2015. It
returns the mu, the parameters of the model, skewness and shape, Ljung-Box and the
Kolmogorov-Smirnov test statistic. The corresponding standard deviations are given in the line
below the values. Finally, the parameters of the distribution of the obtained residuals are
estimated.'

Keywords : ARMA, GARCH, Kolmogorov-Smirnov test, Ljung-Box, estimation

See also : CopDynEst

Author : Verena Weber

Datafile : DataDAXDJTKVW.txt

Output : 'COPretaparch returns tables with the estimations of the mu, the parameters of the model,
skewness and shape, Ljung-Box and the Kolmogorov-Smirnov test statistic for the stocks and the
indices. The corresponding standard deviations are given in the line below the values. The
parameters of the distribution of the obtained residuals are also summarised in a table.'

```


### R Code:
```r
rm(list = ls(all = TRUE))

# please install these packages if necessary
# install.packages("fGarch")
# install.packages("QRM")
# install.packages("copula")

library(fGarch)
library(QRM)
library(copula)

#setwd("C:/...") # please change your working directory

S         = read.table("DataDAXDJTKVW.txt", skip = 6, header = T, sep=";")[, c(2:5)] 
date.time = read.table("DataDAXDJTKVW.txt", sep=";",  skip=7)[, 1] 

#compute log returns
X = S[-1,]
for(i in 1:length(S[1,]))X[,i] = log(S[-1,i]/S[-length(S[,i]),i]) 


feps    = matrix(nrow = dim(X)[1], ncol = dim(X)[2])
eps     = matrix(nrow = dim(X)[1], ncol = dim(X)[2])
eps.squ = matrix(nrow = dim(X)[1], ncol = dim(X)[2])
sigma.t = matrix(nrow = dim(X)[1], ncol = dim(X)[2])

###############################
#### AR(1)-GJR-GARCH(2,1) Fit

#select the indices
sel.pairs = c(1,2) 

params.ind  = matrix(nrow = 2 * dim(X[, sel.pairs])[2], ncol = 14)

for(i in sel.pairs) {
  fit                       = garchFit(~arma(1,0) + aparch(2, 1), delta = 2, 
                              include.delta = FALSE, data = X[, i], 
                              cond.dist = "sged", trace = F)
  eps[, i]                  = fit@residuals
  sigma.t[, i]              = fit@sigma.t
  eps[, i]                  = eps[, i] / sigma.t[, i]   # standardize residuals   
  eps.squ[, i]              = eps[, i]^2 
  params.ind[(2 * i - 1), ] = c(fit@fit$coef, 
                              Box.test(eps[, i], type = "Ljung-Box", lag = 12)$p.value, 
                              Box.test(eps.squ[, i], type = "Ljung-Box", lag = 12)$p.value, 
                              ks.test(eps[, i], "pnorm")$p.value, 
                              ks.test(eps[, i], "psged")$p.value)
  params.ind[(2 * i), ]     = c(fit@fit$matcoef[, 2], NA, NA, NA, NA)
  feps[, i]                 = edf(eps[, i], adjust = TRUE)
}

colnames(params.ind) = c("mu", "ar1", "omega", "alpha1", "alpha2", "gamma1", 
                         "gamma2", "beta1", "skew", "shape", "BL.eps", 
                         "BL.eps.squ","KS.norm", "KS.sged")
rownames(params.ind) = rep(colnames(X[, sel.pairs]), each = 2)

###############################
#### AR(1)-GJR-GARCH(1,1) Fit

#select the stocks
sel.pairs = c(3,4) 

params.sto  = matrix(nrow = 2 * dim(X[, sel.pairs])[2], ncol = 12)

for(i in sel.pairs) {
  fit                       = garchFit(~arma(1,0) + aparch(1, 1), delta = 2, 
                              include.delta = FALSE, data = X[, i], 
                              cond.dist = "sged", trace = F)
  eps[, i]                  = fit@residuals
  sigma.t[, i]              = fit@sigma.t
  eps[, i]                  = eps[, i] / sigma.t[, i]   # standardize residuals   
  eps.squ[, i]              = eps[, i]^2 
  params.sto[(2 * i - 5), ] = c(fit@fit$coef, 
                              Box.test(eps[, i], type = "Ljung-Box", lag = 12)$p.value, 
                              Box.test(eps.squ[, i], type = "Ljung-Box", lag = 12)$p.value, 
                              ks.test(eps[, i], "pnorm")$p.value, 
                              ks.test(eps[, i], "psged")$p.value)
  params.sto[(2 * i - 4), ] = c(fit@fit$matcoef[, 2], NA, NA, NA, NA)
  feps[, i]                 = edf(eps[, i], adjust = TRUE)
}

colnames(params.sto) = c("mu", "ar1", "omega", "alpha1", "gamma1", "beta1", "skew",
                         "shape", "BL.eps", "BL.eps.squ", "KS.norm", "KS.sged")
rownames(params.sto) = rep(colnames(X[, sel.pairs]), each = 2)

#############################################################
### Estimate parameters of distribution of obtained residuals

params.margins = matrix(NA, nrow = dim(eps)[2], ncol = 4)

for (i in 1:dim(eps)[2]) {
  sgedFit             = sgedFit(eps[ ,i])
  params.margins[i, ] = sgedFit$par
}

colnames(params.margins) = c("mean", "sd", "shape", "skew")
rownames(params.margins) = colnames(X)
colnames(eps)            = colnames(X)

#############
### KS test

params.ind[1, 14] = ks.test(eps[, 1], "psged", 0 , 1, round(params.margins[1, 3], digits = 2),
                    round(params.margins[1, 4], digits = 1))$p.value
params.ind[3, 14] = ks.test(eps[, 2], "psged", 0 , 1, round(params.margins[2, 3], digits = 2),
                    round(params.margins[2, 4], digits = 1))$p.value
params.sto[1, 12] = ks.test(eps[, 3], "psged", 0 , 1, round(params.margins[3, 3], digits = 2),
                    round(params.margins[3, 4], digits = 1))$p.value
params.sto[3, 12] = ks.test(eps[, 4], "psged", 0 , 1, round(params.margins[4, 3], digits = 2),
                    round(params.margins[4, 4], digits = 1))$p.value

params.ind
params.sto
params.margins 
```
