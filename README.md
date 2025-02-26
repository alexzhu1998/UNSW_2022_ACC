# Actuarial Theory and Practice A Assignment

By Aidan Yeoh, Alex Zhu, Annie Zhu, Matthew Winfred, Rosie Tao

1.  Data Cleaning Steps
2.  Modelling Steps
3.  Economic Impact Steps
4.  Conclusion

``` r
# LINEAR REGRESSION

cols_to_remove <- c("Pos_new","Player","Nation","League","Squad")
pos_levels <- c("MF","DF","FW")

for (level in pos_levels) {
    eval(call("<-",paste0(level,"_nonRFL"),
              nonRFL %>% filter(Pos_new == level) %>% select(-all_of(cols_to_remove))))
    eval(call("<-",paste0(level,"_RFL"),
              RFL %>% filter(Pos_new == level) %>% select(-all_of(cols_to_remove))))
    eval(call("<-",paste0(level,"_nonRFL_mod"),
              glm(Annualized_Salary ~ ., data = eval(str2lang(paste0(level,"_nonRFL"))))))
    
    Predicted_Sal<- predict(eval(str2lang(paste0(level,"_nonRFL_mod"))),newdata = eval(str2lang(paste0(level,"_RFL"))))
    eval(call("<-",paste0(level,"_RFL"),
              cbind(get(paste0(level,"_RFL")),Predicted_Sal)))
    
    Diff <- eval(str2lang(paste0(level,"_RFL")))["Predicted_Sal"] - eval(str2lang(paste0(level,"_RFL")))["Annualized_Salary"]
    names(Diff) <- "Diff"
    eval(call("<-",paste0(level,"_RFL"),
              cbind(get(paste0(level,"_RFL")),Diff)))
    
    
}


mean(FW_nonRFL_mod$residuals^2)
```

    ## [1] 2.946836e+13

``` r
mean((DF_nonRFL_mod$residuals^2))
```

    ## [1] 2.483974e+13

``` r
mean(MF_nonRFL_mod$residuals^2)
```

    ## [1] 3.381205e+13

``` r
plot(MF_RFL$Annualized_Salary,MF_RFL$Predicted_Sal, main= "MF RFL")
```

![](README_files/figure-markdown_github/linear_regression_models-1.png)

``` r
plot(DF_RFL$Annualized_Salary,DF_RFL$Predicted_Sal, main= "DF RFL")
```

![](README_files/figure-markdown_github/linear_regression_models-2.png)

``` r
plot(FW_RFL$Annualized_Salary,FW_RFL$Predicted_Sal, main= "FW RFL")
```

![](README_files/figure-markdown_github/linear_regression_models-3.png)

``` r
MF_RFL %>% arrange(Diff,descending = T)
DF_RFL %>% arrange(Diff,descending = T)
FW_RFL %>% arrange(Diff,descending = T)


## DO NOT CHANGE COLUMN ORDER -> CHANGING COLUMN ORDER WILL REQUIRE THE FOLLOWING CODE TO BE MODIFIED ##

#Fit boosting model on annualised salary dataset with non RFL football leagues and ALL predictors
set.seed(1)
colnames(df)[c(1,2,3,4,5,71)]

gbmFit.param <- gbm(Annualized_Salary ~., data = df[df['League'] != "RFL",-c(1,2,3,4,5,71)], distribution = "gaussian", cv.fold = 10, n.trees = 3000, interaction.depth = 1, shrinkage = 0.01)
gbmFit.param

min <- which.min(gbmFit.param$cv.error)
min
gbm.perf(gbmFit.param, method = "cv")

gbmFit <- gbm(Annualized_Salary ~., df[df['League'] != "RFL",-c(1,2,3,4,5,71)], distribution = "gaussian", n.trees = min, interaction.depth = 1, shrinkage = 0.01)

summary(gbmFit)

gbm.predict = predict(gbmFit, newdata = df[,-c(1,2,3,4,5,70,71)], n.trees = min, type = "response")


plot(gbm.predict[(df['League'] != "RFL")], df$Annualized_Salary[(df['League'] != "RFL")])
plot(gbm.predict[(df['League'] == "RFL")], df$Annualized_Salary[(df['League'] == "RFL")])

plot(gbm.predict[(df['Nation'] != "Rarita")], df$Annualized_Salary[(df['Nation'] != "Rarita")])
plot(gbm.predict[(df['Nation'] == "Rarita")], df$Annualized_Salary[(df['Nation'] == "Rarita")])

#New dataframe with correlated vairables removed + identifiable attributes
cor_df_merge <- cbind(cor_df,df[,c("Player","Nation","Pos_new","League","Squad")])
stopifnot((nrow(cor_df_merge) == 5500) && (length(colnames(cor_df_merge))== 23)) # Added check 
#MF model
gbmFit.param_MF <- gbm(Annualized_Salary ~., data = cor_df_merge[(cor_df_merge['League'] != "RFL") & (cor_df_merge['Pos_new'] == "MF"),-c(19,20,21,22,23)], distribution = "gaussian", cv.fold = 10, n.trees = 3000, interaction.depth = 1, shrinkage = 0.01)
gbmFit.param_MF

min_MF <- which.min(gbmFit.param_MF$cv.error)
min_MF
gbm.perf(gbmFit.param_MF, method = "cv")

gbmFit_MF <- gbm(Annualized_Salary ~., cor_df_merge[(cor_df_merge['League'] != "RFL") & (cor_df_merge['Pos_new'] == "MF"),-c(19,20,21,22,23)], distribution = "gaussian", n.trees = min_MF, interaction.depth = 1, shrinkage = 0.01)

summary(gbmFit_MF)
gbm.predict_MF = predict(gbmFit_MF, newdata = cor_df_merge[,-c(17, 19,20,21,22,23)], n.trees = min_MF, type = "response")

#Comparing actual vs expected in MF model
hist(gbm.predict_MF[(cor_df_merge['Pos_new'] == "MF") & (df['League'] != "RFL")])
hist(df$Annualized_Salary[(df['League'] != "RFL") & (df['Pos_new'] == "MF")], breaks = 20)

plot(gbm.predict_MF[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "MF")], df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "MF")])
plot(gbm.predict_MF[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "MF")], df$Annualized_Salary[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "MF")])

colnames(cor_df_merge)[c(17,19,20,21,22,23)]

mean((gbm.predict_MF[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "MF")]-df$Annualized_Salary[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "MF")])^2)

#DF model
gbmFit.param_DF <- gbm(Annualized_Salary ~., data = cor_df_merge[(cor_df_merge['League'] != "RFL") & (cor_df_merge['Pos_new'] == "DF"),-c(19,20,21,22,23)], distribution = "gaussian", cv.fold = 10, n.trees = 3000, interaction.depth = 1, shrinkage = 0.01)
gbmFit.param_DF

min_DF <- which.min(gbmFit.param_DF$cv.error)
min_DF
gbm.perf(gbmFit.param_DF, method = "cv")

gbmFit_DF <- gbm(Annualized_Salary ~., cor_df_merge[(cor_df_merge['League'] != "RFL") & (cor_df_merge['Pos_new'] == "DF"),-c(19,20,21,22,23)], distribution = "gaussian", n.trees = min_DF, interaction.depth = 1, shrinkage = 0.01)

summary(gbmFit_DF)

gbm.predict_DF = predict(gbmFit_DF, newdata = cor_df_merge[,-c(17, 19,20,21,22,23)], n.trees = min_DF, type = "response")

#Comparing actual vs expected in DF model
hist(gbm.predict_DF[(cor_df_merge['Pos_new'] == "DF") & (df['League'] != "RFL")])
hist(df$Annualized_Salary[(df['League'] != "RFL") & (df['Pos_new'] == "DF")], breaks = 20)

plot(gbm.predict_DF[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "DF")], df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "DF")])
plot(gbm.predict_DF[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "DF")], df$Annualized_Salary[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "DF")])

mean((gbm.predict_DF[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "DF")]-df$Annualized_Salary[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "DF")])^2)

#FW model
gbmFit.param_FW <- gbm(Annualized_Salary ~., data = cor_df_merge[(cor_df_merge['League'] != "RFL") & (cor_df_merge['Pos_new'] == "FW"),-c(19,20,21,22,23)], distribution = "gaussian", cv.fold = 10, n.trees = 3000, interaction.depth = 1, shrinkage = 0.01)
gbmFit.param_FW

min_FW <- which.min(gbmFit.param_FW$cv.error)
min_FW
gbm.perf(gbmFit.param_FW, method = "cv")

gbmFit_FW <- gbm(Annualized_Salary ~., cor_df_merge[(cor_df_merge['League'] != "RFL") & (cor_df_merge['Pos_new'] == "FW"),-c(19,20,21,22,23)], distribution = "gaussian", n.trees = min_FW, interaction.depth = 1, shrinkage = 0.01)

summary(gbmFit_FW)

gbm.predict_FW = predict(gbmFit_FW, newdata = cor_df_merge[,-c(17,19,20,21,22,23)], n.trees = min_FW, type = "response")

#Comparing actual vs expected in FW model
hist(gbm.predict_FW[(cor_df_merge['Pos_new'] == "FW") & (df['League'] != "RFL")])
hist(df$Annualized_Salary[(df['League'] != "RFL") & (df['Pos_new'] == "FW")], breaks = 20)

plot(gbm.predict_FW[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "FW")], df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "FW")])
plot(gbm.predict_FW[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "FW")], df$Annualized_Salary[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "FW")])

mean((gbm.predict_FW[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "FW")]-df$Annualized_Salary[(df['League'] != "RFL") & (cor_df_merge['Pos_new'] == "FW")])^2)

#GK model
gbmFit.param_GK <- gbm(Annualized_Salary ~., data = gk_df[(gk_df['League'] != "RFL"),-c(16,17,18,19,20)], distribution = "gaussian", cv.fold = 10, n.trees = 3000, interaction.depth = 1, shrinkage = 0.01)
gbmFit.param_GK

min_GK <- which.min(gbmFit.param_GK$cv.error)
min_GK
gbm.perf(gbmFit.param_GK, method = "cv")

gbmFit_GK <- gbm(Annualized_Salary ~., data = gk_df[(gk_df['League'] != "RFL"),-c(16,17,18,19,20)], distribution = "gaussian", cv.fold = 10, n.trees = min_GK, interaction.depth = 1, shrinkage = 0.01)
summary(gbmFit_GK)

gbm.predict_GK = predict(gbmFit_GK, newdata = gk_df[,-c(15,16,17,18,19,20)], n.trees = min_GK, type = "response")

#Comparing actual vs expected in GK model
plot(gbm.predict_GK[(gk_df['League'] == "RFL")], gk_df$Annualized_Salary[(gk_df['League'] == "RFL")])
plot(gbm.predict_GK[(gk_df['League'] != "RFL")], gk_df$Annualized_Salary[(gk_df['League'] != "RFL")])

mean((gbm.predict_GK[(gk_df['League'] != "RFL")]-gk_df$Annualized_Salary[(gk_df['League'] != "RFL")])^2)

#Actual salary histograms
hist(df$Annualized_Salary[(df['League'] != "RFL") & (df['Pos_new'] == "DF")], breaks = 20)
hist(df$Annualized_Salary[(df['League'] != "RFL") & (df['Pos_new'] == "FW")], breaks = 20)
hist(df$Annualized_Salary[(df['League'] != "RFL") & (df['Pos_new'] == "MF")], breaks = 20)
hist(gk_df$Annualized_Salary[(gk_df['League'] != "RFL")], breaks = 20)


## Apply model to tournament data, obtain national team "scores"

gbm.tourn_MF = predict(gbmFit_MF, newdata = cor_tourn_merge[,-c(17,19,20,21,22,23)], n.trees = min_MF, type = "response")
gbm.tourn_DF = predict(gbmFit_DF, newdata = cor_tourn_merge[,-c(17,19,20,21,22,23)], n.trees = min_DF, type = "response")
gbm.tourn_FW = predict(gbmFit_FW, newdata = cor_tourn_merge[,-c(17,19,20,21,22,23)], n.trees = min_FW, type = "response")
gbm.tourn_GK = predict(gbmFit_GK, newdata = gk_tourn_df[,-c(1,2,3,4)], n.trees = min_GK, type = "response")

MF_tourn <- cbind(select(cor_tourn_merge, c('Player','Nation','Pos_new')), gbm.tourn_MF)
MF_tourn_df <- MF_tourn %>%
    filter(Pos_new == "MF")

MF_stats <- MF_tourn_df %>%
    group_by(Nation) %>%
    summarise(MF_Score = mean(gbm.tourn_MF))
MF_stats %>% arrange(MF_Score,descending = T)

DF_tourn <- cbind(select(cor_tourn_merge, c('Player','Nation','Pos_new')), gbm.tourn_DF)
DF_tourn_df <- DF_tourn %>%
    filter(Pos_new == "DF")

DF_stats <- DF_tourn_df %>%
    group_by(Nation) %>%
    summarise(DF_Score = mean(gbm.tourn_DF))
DF_stats %>% arrange(DF_Score,descending = T)

FW_tourn <- cbind(select(cor_tourn_merge, c('Player','Nation','Pos_new')), gbm.tourn_FW)
FW_tourn_df <- FW_tourn %>%
    filter(Pos_new == "FW")

FW_stats <- FW_tourn_df %>%
    group_by(Nation) %>%
    summarise(FW_Score = mean(gbm.tourn_FW))
FW_stats %>% arrange(FW_Score,descending = T)

GK_tourn_df <- cbind(gk_tourn_df[,c(1,2)], gbm.tourn_GK)
GK_stats <- GK_tourn_df %>%
    group_by(Nation) %>%
    summarise(GK_Score = mean(gbm.tourn_GK))
GK_stats %>% arrange(GK_Score, descending = T)

colnames(PLAYER_tourn_res_2021)[2] <- "Nation"
team_stats <- merge(merge(merge(merge(MF_stats, DF_stats), FW_stats), GK_stats),PLAYER_tourn_res_2021)
total_score <- team_stats$FW_Score*2/11 + team_stats$MF_Score*4/11 + team_stats$DF_Score*4/11 + team_stats$GK_Score*1/11

team_stats <- cbind(team_stats,total_score)
team_stats <- team_stats %>% arrange(`2021 Tournament Place`, descending = T)

plot(team_stats$`2021 Tournament Place`, team_stats$DF_score)

# write.csv(team_stats,"data/match_model_data.csv")
```

``` r
model_data <- read.csv("data/match_model.csv")

model_data$Outcome <-ifelse(model_data$Outcome == 'Win',1,0)


#model_data$Outcome[model_data$Outcome == "Win"] <- 1
#model_data$Outcome[model_data$Outcome == "Lose"] <- 0
model_data$Outcome <- as.numeric(model_data$Outcome)
summary(model_data)
model_data <- as.data.frame(model_data)

summary(select(model_data, -c("Rank_A","Rank_B","Name_A","Name_B")))

glm_mod <- glm(Outcome ~ ., data = select(model_data, -c("Rank_A","Rank_B","Name_A","Name_B")), family = binomial())
summary(glm_mod)

#Fit gradient booster to link scores with match outcomes
gbmMatch_param <- gbm(Outcome ~., data = model_data[,-c(2,3,4,5)], distribution = "bernoulli", cv.fold = 10, n.trees = 3000, interaction.depth = 1, shrinkage = 0.01)
gbmMatch_param


min_match_param <- which.min(gbmMatch_param$cv.error)
min_match_param
gbm.perf(gbmMatch_param, method = "cv")

gbm_match <- gbm(Outcome ~., data = model_data[,-c(2,3,4,5)], distribution = "bernoulli", n.trees = min_match_param, interaction.depth = 1, shrinkage = 0.01)

summary(gbm_match)

gbm.match.predict = predict(gbm_match, newdata = model_data[,-c(1,2,3,4,5)], n.trees = min_match_param, type = "response")

mean(gbm.match.predict[1:246])
mean(gbm.match.predict[247:492])





# Raritian players updated table ------------------------------------------
column.names <- c('Player','Annualized_Salary','Expected_Salary', 'Pos_new', 'Salary_Ratio')
select.quantile <- 0

# gbm.vector <- c(gbm.predict_DF,gbm.predict_FW,gbm.predict_GK,gbm.predict_MF)
player.names <- cor_df_merge[,c('Player','Annualized_Salary','Pos_new','Nation')]

#Field players
player.salary <- cbind(player.names, gbm.predict_DF)
player.salary <- cbind(player.salary, gbm.predict_MF)
player.salary <- cbind(player.salary, gbm.predict_FW)

#Filter out RFL player

rarita.players <- player.salary %>% filter(Nation == 'Rarita')

rarita.mf <- rarita.players %>%
    filter(Pos_new == 'MF') %>%
    filter(quantile(Annualized_Salary, select.quantile) < Annualized_Salary)%>%
    select(Player, Annualized_Salary, gbm.predict_MF, Pos_new)%>%
    mutate(salary.ratio = gbm.predict_MF/Annualized_Salary)%>%
    arrange(desc(salary.ratio))

rarita.df <- rarita.players %>%
    filter(Pos_new == 'DF') %>%
    filter(quantile(Annualized_Salary, select.quantile) < Annualized_Salary)%>%
    select(Player, Annualized_Salary, gbm.predict_DF, Pos_new)%>%
    mutate(salary.ratio = gbm.predict_DF/Annualized_Salary)%>%
    arrange(desc(salary.ratio))

rarita.fw <- rarita.players %>%
    filter(Pos_new == 'FW') %>%
    filter(quantile(Annualized_Salary, select.quantile) < Annualized_Salary) %>%
    select(Player, Annualized_Salary, gbm.predict_FW, Pos_new)%>%
    mutate(salary.ratio = gbm.predict_FW/Annualized_Salary)%>%
    arrange(desc(salary.ratio))

#Goalkeepers
goalkeepers <- gk_df[,c('Player','Annualized_Salary','Nation')]
gk.salary <- cbind(goalkeepers, gbm.predict_GK)
rarita.gk <- gk.salary %>%
    filter(Nation == 'Rarita')%>%
    filter(quantile(Annualized_Salary, select.quantile) < Annualized_Salary)%>%
    select(Player, Annualized_Salary, gbm.predict_GK)%>%
    mutate(salary.ratio = gbm.predict_GK/Annualized_Salary)%>%
    arrange(desc(salary.ratio))



colnames(rarita.df) <- column.names
colnames(rarita.mf) <- column.names
colnames(rarita.fw) <- column.names
rarita.gk <- cbind(rarita.gk, Pos_new = rep(c("GK")))
rarita.gk <- rarita.gk[,c(1,2,3,5,4)]
colnames(rarita.gk) <- column.names

#Make football team
#pick 3 goalkeepers, 7 df, 7 mf, 5fw

national.team <- rarita.gk[1:3,]
national.team <- rbind(national.team, rarita.df[1:7,])
national.team <- rbind(national.team, rarita.mf[1:7,])
national.team <- rbind(national.team, rarita.fw[1:5,])




#PDP graphs
par.df.DF <- partial(gbmFit_DF, pred.var = c('Expected_xG'), n.trees = min_DF)
par.df.DF <- partial(gbmFit_DF, pred.var = c('xA'), n.trees = min_DF)
par.df.DF <- partial(gbmFit_DF, pred.var = c('Tackles_Tkl'), n.trees = min_DF)
autoplot(par.df.DF, contour = TRUE)

total_score <- team_stats$FW_Score*2/11 + team_stats$MF_Score*4/11 + team_stats$DF_Score*4/11 + team_stats$GK_Score*1/11


national.team.stats <- national.team %>%
    group_by(Pos_new) %>%
    summarise(Score = mean(Expected_Salary))



national.team.stats[1,2]*1/11+ national.team.stats[2,2]*4/11 + national.team.stats[3,2]*4/11 + national.team.stats[4,2]*2/11
15151245*1/11+ 19606225*4/11 + 22892307*4/11 + 24088798*2/11


final.national.team <- national.team.stats%>%
    add_row(Pos_new = "Total", Score = (national.team.stats[1,2]*1/11 
                                        + national.team.stats[2,2]*4/11 + national.team.stats[3,2]*4/11 + 
                                            national.team.stats[4,2]*2/11))

national.team.matchups <- read.csv("data/bad_team.csv")


national.team.predict = predict(gbm_match, newdata = national.team.matchups[,-c(1,2)], n.trees = min_match_param, type = "response")

national.team.matchups <- cbind(national.team.matchups, Probs = national.team.predict)
```

``` r
#Our team vs [18,23],[12,17],[6,11],[1,5]

set.seed(1)
#Probability that our team is in the top 10 at least once within 5 years
prob_top10_5yrs <- c()
for (i in 1:1000) {
    #successful outcome
    sim_counter <- 0
    #Calculate a single probability
    for (j in 1:1000) {
        win_two_match_prob <- national.team.matchups[floor(runif(5, min = 18, max = 24)),"Probs"]*national.team.matchups[floor(runif(5, min = 12, max = 17)),"Probs"]
        #How many times I become top 10 in 5 yrs
        count <- 0
        
        for (k in 1:5) {
            count <- count + rbinom(1, 1, win_two_match_prob[k])
        }
        
        if (count >= 1) {
            sim_counter <- sim_counter + 1
        }
    }
    
    prob_top10_5yrs[i] <- sim_counter/1000
}
hist(prob_top10_5yrs)

prob_top10_5yrs.df <- data.frame(probs = prob_top10_5yrs)

#CI of top 10
mean(prob_top10_5yrs) - qnorm(0.975, 0, 1) * sd(prob_top10_5yrs) / sqrt(1000)
mean(prob_top10_5yrs) + qnorm(0.975, 0, 1) * sd(prob_top10_5yrs) / sqrt(1000)


ggplot(prob_top10_5yrs.df)+
    geom_histogram(aes(x = probs, y = ..density..), color = "black", fill="#5662d1", bins = 30)+
    labs(x = "Probability of attaining FSA Top 10 in 5 years", y = "Density", title = "Distribution of Simulated Probability", subtitle = "FSA Top 10 in 5 years")+
    theme_bw() +
    theme(axis.text=element_text(size=9.5), axis.title=element_text(size=13, face = "bold"), plot.title = element_text(size=16, face = "bold"), plot.subtitle=element_text(size=13))

set.seed(1)
#Probability that our team is in the top 10 for the majority of the time within 5 years
prob_top10_5yrs_majority <- c()
for (i in 1:1000) {
    #successful outcome
    sim_counter <- 0
    #Calculate a single probability
    for (j in 1:1000) {
        win_two_match_prob <- national.team.matchups[floor(runif(5, min = 18, max = 24)),"Probs"]*national.team.matchups[floor(runif(5, min = 12, max = 18)),"Probs"]
        #How many times I become top 10 in 5 yrs
        count <- 0
        
        for (k in 1:5) {
            count <- count + rbinom(1, 1, win_two_match_prob[k])
        }
        
        if (count >= 3) {
            sim_counter <- sim_counter + 1
        }
    }
    
    prob_top10_5yrs_majority[i] <- sim_counter/1000
}
hist(prob_top10_5yrs_majority)



set.seed(1)
#Probability that our team wins the championship at least once within 10 years
prob_win_10yrs <- c()
for (i in 1:1000) {
    #successful outcome
    sim_counter <- 0
    #Calculate a single probability
    for (j in 1:1000) {
        win_prob <- national.team.matchups[floor(runif(10, min = 18, max = 24)),"Probs"]*national.team.matchups[floor(runif(10, min = 12, max = 18)),"Probs"]*national.team.matchups[floor(runif(10, min = 6, max = 12)),"Probs"]*national.team.matchups[floor(runif(10, min = 1, max = 6)),"Probs"]
        #How many times I win
        count <- 0
        
        for (k in 1:10) {
            count <- count + rbinom(1, 1, win_prob[k])
        }
        
        if (count >= 1) {
            sim_counter <- sim_counter + 1
        }
    }
    
    prob_win_10yrs[i] <- sim_counter/1000
}
hist(prob_win_10yrs)

#CI of winning prob
mean(prob_win_10yrs) - qnorm(0.975, 0, 1) * sd(prob_win_10yrs) / sqrt(1000)
mean(prob_win_10yrs) + qnorm(0.975, 0, 1) * sd(prob_win_10yrs) / sqrt(1000)

prob_win_10yrs.df <- data.frame(probs = prob_win_10yrs)


ggplot(prob_win_10yrs.df)+
    geom_histogram(aes(x = probs, y = ..density..), color = "black", fill="#5662d1", bins = 30)+
    labs(x = "Probability of winning FSA Championship in 10 years", y = "Density", title = "Distribution of Simulated Probability", subtitle = "FSA Championship in 10 years")+
    theme_bw() +
    theme(axis.text=element_text(size=9.5), axis.title=element_text(size=13, face = "bold"), plot.title = element_text(size=16, face = "bold"), plot.subtitle=element_text(size=13))


#Probability thresholds over time - monitoring performance for the at least one win within 10 years
set.seed(1)
prob_win_10yrs_benchmark <- c()
for (i in 1:10) {
    sim_counter <- 0
    #Calculate a single probability benchmark
    for (j in 1:1000) {
        win_prob <- national.team.matchups[floor(runif(i, min = 18, max = 24)),"Probs"]*national.team.matchups[floor(runif(i, min = 12, max = 18)),"Probs"]*national.team.matchups[floor(runif(i, min = 6, max = 12)),"Probs"]*national.team.matchups[floor(runif(i, min = 1, max = 6)),"Probs"]
        #How many times I win
        count <- 0
        
        for (k in 1:i) {
            count <- count + rbinom(1, 1, win_prob[k])
        }
        
        if (count >= 1) {
            sim_counter <- sim_counter + 1
        }
    } 
    prob_win_10yrs_benchmark[i] <- sim_counter/1000
}

prob_win_10yrs_benchmark <- prob_win_10yrs_benchmark - (prob_win_10yrs_benchmark[10]-0.7)




#Probability thresholds over time - monitoring performance for the at least one top 10 within 5 years
set.seed(1)
prob_top10_5yrs_benchmark <- c()
for (i in 1:5) {
    sim_counter <- 0
    #Calculate a single probability benchmark
    for (j in 1:1000) {
        win_prob <- national.team.matchups[floor(runif(i, min = 18, max = 24)),"Probs"]*national.team.matchups[floor(runif(i, min = 12, max = 18)),"Probs"]
        #How many times I become top 10 in 5 yrs
        count <- 0
        
        for (k in 1:i) {
            count <- count + rbinom(1, 1, win_prob[k])
        }
        
        if (count >= 1) {
            sim_counter <- sim_counter + 1
        }
    } 
    prob_top10_5yrs_benchmark[i] <- sim_counter/1000
}

prob_top10_5yrs_benchmark <- prob_top10_5yrs_benchmark - (prob_top10_5yrs_benchmark[5]-0.85)


ten.year.bm <- data.frame(x = seq(2022,2031),
                           y = sort(prob_win_10yrs_benchmark, TRUE))

five.year.bm <- data.frame(x = seq(2022,2026),
                           y = sort(prob_top10_5yrs_benchmark, TRUE))
```

``` r
#Set xend and yend
ten.year.bm$xend <- seq(2023,2032)
ten.year.bm$yend <- ten.year.bm$y

five.year.bm$xend <- seq(2023,2027)
five.year.bm$yend <- five.year.bm$y

ggplot(ten.year.bm)+
    geom_segment(aes(x = x, y = y, xend = xend, yend = yend), color = 'royalblue', cex = 1)+
    labs(title = "Competitive Benchmark Probabilities", subtitle = 'Probability of Winning FSA over remaining years',
         x = "Current Year", y = "Probability")+
    scale_y_continuous(breaks = seq(0,1,0.1), limits = c(0,1))+
    scale_x_continuous(breaks = seq(2022,2032,1), limits = c(2022,2032))+
    theme_bw()+
    theme(axis.text=element_text(size=9.5), axis.title=element_text(size=13, face = "bold"), plot.title = element_text(size=14, face = "bold"))
    

ggplot(five.year.bm)+
    geom_segment(aes(x = x, y = y, xend = xend, yend = yend), color = 'royalblue', cex = 1)+
    labs(title = "Competitive Benchmark Probabilities", subtitle = 'Probability of Placing in the Top 10 over remaining years',
         x = "Current Year", y = "Probability")+
    scale_y_continuous(breaks = seq(0,1,0.1), limits = c(0,1))+
    scale_x_continuous(breaks = seq(2022,2027,1), limits = c(2022,2027))+
    theme_bw()+
    theme(axis.text=element_text(size=9.5), axis.title=element_text(size=13, face = "bold"), plot.title = element_text(size=14, face = "bold"))
    

#ggplot(benchmark.df)+
#    geom_segment(aes(x = x, y = ten_year, xend=xend, yend = yend_ten), show.legend = TRUE)+
#   geom_segment(aes(x = x, y = five_year, xend=xend, yend = yend_five), color = 'red', show.legend = TRUE)+
#    labs(title = "Competitive Benchmarks", x = "Year", y = "Probability")+
#    scale_x_continuous(breaks = seq(1,10))+
#    scale_y_continuous(breaks = seq(0,1,0.1))
#    theme(legend.title = "Objectives", position = "bottom")+
#    theme_bw()



#Cost of league (player salaries) - ECON model
sum(cor_df$Annualized_Salary[(df$League == "RFL") & (df$Year == "2020")]) + sum(gk_df$Annualized_Salary[(gk_df$League == "RFL")])/2

#Plots for player selection
#MF
MF_plot_data <- data.frame(cbind(Standardised_Salary = gbm.predict_MF[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "MF")], Annualised_Salary = df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "MF")]))
MF_select <- MF_plot_data[
    (gbm.predict_MF[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "MF")]/df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "MF")]>4.4),]

ggplot(MF_plot_data, aes(x = Annualised_Salary, y = Standardised_Salary)) +
    geom_point()+
    theme_bw()+
    geom_smooth(method=lm, se = FALSE, formula=y~x-1)+
    geom_encircle(data = MF_select, color = "red", size = 2, expand = 0.03)+
    labs(x = "Annualised Salary (∂)", y = "Standardised Salary (∂)", title = "Relationship between Standardised and Annualised Salary", subtitle = "RFL MF Players")+
    theme(axis.text=element_text(size=9.5), axis.title=element_text(size=13, face = "bold"), plot.title = element_text(size=14, face = "bold"))+
    scale_y_continuous(labels = scales::unit_format(unit = "M", scale = 1e-6))+
    scale_x_continuous(labels = scales::unit_format(unit = "M", scale = 1e-6))

#DF
DF_plot_data <- data.frame(cbind(Standardised_Salary = gbm.predict_DF[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "DF")], Annualised_Salary = df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "DF")]))
DF_select <- DF_plot_data[(gbm.predict_DF[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "DF")]/df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "DF")] > 4.35),]

ggplot(DF_plot_data, aes(x = Annualised_Salary, y = Standardised_Salary)) +
    geom_point()+
    theme_bw()+
    geom_smooth(method=lm, se = FALSE, formula=y~x-1)+
    geom_encircle(data = DF_select, color = "red", size = 2, expand = 0.03)+
    labs(x = "Annualised Salary (∂)", y = "Standardised Salary (∂)", title = "Relationship between Standardised and Annualised Salary", subtitle = "RFL DF Players")+
    theme(axis.text=element_text(size=9.5), axis.title=element_text(size=13, face = "bold"), plot.title = element_text(size=14, face = "bold"))+
    scale_y_continuous(labels = scales::unit_format(unit = "M", scale = 1e-6))+
    scale_x_continuous(labels = scales::unit_format(unit = "M", scale = 1e-6))

#FW
FW_plot_data <- data.frame(cbind(Standardised_Salary = gbm.predict_FW[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "FW")], Annualised_Salary = df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "FW")]))
FW_select <- FW_plot_data[(gbm.predict_FW[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "FW")]/df$Annualized_Salary[(df['League'] == "RFL") & (cor_df_merge['Pos_new'] == "FW")] > 4.41),]

ggplot(FW_plot_data, aes(x = Annualised_Salary, y = Standardised_Salary)) +
    geom_point()+
    theme_bw()+
    geom_smooth(method=lm, se = FALSE, formula=y~x-1)+
    geom_encircle(data = FW_select, color = "red", size = 2, expand = 0.03)+
    labs(x = "Annualised Salary (∂)", y = "Standardised Salary (∂)", title = "Relationship between Standardised and Annualised Salary", subtitle = "RFL FW Players")+
    theme(axis.text=element_text(size=9.5), axis.title=element_text(size=13, face = "bold"), plot.title = element_text(size=14, face = "bold"))+
    scale_y_continuous(labels = scales::unit_format(unit = "M", scale = 1e-6))+
    scale_x_continuous(labels = scales::unit_format(unit = "M", scale = 1e-6))

#GK
GK_plot_data <- data.frame(cbind(Standardised_Salary = gbm.predict_GK[(df['League'] == "RFL")], Annualised_Salary = gk_df$Annualized_Salary[(df['League'] == "RFL")]))
GK_select <- GK_plot_data[(gbm.predict_GK[(df['League'] == "RFL")]/gk_df$Annualized_Salary[(df['League'] == "RFL")] > 1),]

ggplot(GK_plot_data, aes(x = Annualised_Salary, y = Standardised_Salary)) +
    geom_point()+
    theme_bw()+
    geom_smooth(method=lm, se = FALSE, formula=y~x-1)+
    geom_encircle(data = GK_select, color = "red", size = 2, expand = 0.03)+
    labs(x = "Annualised Salary (∂)", y = "Standardised Salary (∂)", title = "Relationship between Standardised and Annualised Salary", subtitle = "RFL GK Players")+
    theme(axis.text=element_text(size=9.5), axis.title=element_text(size=13, face = "bold"), plot.title = element_text(size=14, face = "bold"))+
    scale_y_continuous(labels = scales::unit_format(unit = "M", scale = 1e-6))+
    scale_x_continuous(labels = scales::unit_format(unit = "M", scale = 1e-6))
```
