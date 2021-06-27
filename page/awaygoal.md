# The away goal rule was not fair enough.

This week, it was announced that UEFA will abolish the "away goal" rule for Champions League games. This rule was introduced in 1965, when the Champions League was called the Europa Cup 1, and was meant to give teams some extra motivation to score during away games in either a playoff or knockout round. After two legs of football matches, if the aggregate score was a tie, the team with the most away goals would win. This rule also counted in extra time, the extra 30 minutes played in the second leg of the match after a tie with no winner, so that if the away team would score during this period, the home team would need to score at least *twice*. This last aspect of the rule has often been framed as particularly unfair for the home team, and I think it is one of the main reasons it was abolished. Here, I will show that this was a mistake, but also how this mistake can be salvaged without reintroducing the rule!

Over the first 180 minutes of football, the away goal rule is not unfair: both teams get 90 minutes to take advantage of it. However, after regular time, something interesting happens. The home team gets to play 30 more minutes in their own stadium, and the away team gets an extra 30 minutes of 'away goal advantage'. In order to truly figure out whether the away goal rule is unfair, we need to simply need to figure out how these two advantages stack up against each other? Most people in the business seem to think that the advantage goes to the away team. In the end, their goals count more than their opponent's goals. If this is truly the case, then we should find that *the away team wins  more during overtime*.

But is this really the case? The great thing is that we do not have to rely on opinion anymore, we can simply look at the data. So, I analyzed data all games from the Europa Cup 1 and Champions Leauge that went into overtime since the rule has been introduced in 1965. The hypothesis is simple: if the away goal rule is unfair to the home team in overtime, then the away team should win games that go to overtime more (than 50%). (I decided to not analyze data from the last two seasons, since I suspect that COVID has had a huge, and unrelated, effect on the home-field advantage.)

The results were very surprising, and speak very much against this rule change! Across all games played so far, there is an overwhelming tendency for the away team to **lose** in extra time: 63% of games our won by the home team after regular time. So, home teams win most of the time, *even with the away goal rule in place*. This clearly shows that, if anything, the away goal rule was not punishing enough.

You can see the data for yourself in the figure below (and you can find all the code for this analysis on my GitHub profile). Of course, there seems to be somewhat of a downward trend over time, but even in the last decade, there is a clear (60%) advantage for the home team during overtime. Now that this rule has been abolished, the home team's advantage will be even greater during overtime! That's not fair at all. The home team was already at an advantage, so taking away this rule will benefit them even more.

Luckily, I discovered an easy fix, which I will tell you about below!



```python
import pandas as pd
import numpy as np
import scipy.stats as stats
from scipy.optimize import minimize
from statsmodels.stats.descriptivestats import sign_test
import math
import os
import matplotlib.pyplot as plt
import seaborn as sns
import matplotlib.ticker as mtick

# load data
dirs = [f for f in os.listdir('data') if not f.startswith('.')]

data = pd.DataFrame()

for i in range(0,len(dirs)) :
    tempdata = pd.read_csv('data/'+dirs[i]+'/champs.csv')
    tempdata = tempdata[~tempdata['ET'].isnull().values]
    tempdata['year'] = int(dirs[i][0:4:1])
    tempdata = tempdata[tempdata["Round"]!="Final"]
    data = data.append(tempdata)

remaining = pd.read_csv('remaining_games.csv')
data = data.append(remaining)

data["home_goals"] = ""
data["away_goals"] = ""
data["penalties"] = ""
data["home_wins"] = np.nan
data["sign"] = np.nan
data.reset_index(inplace=True)

nrdatasets=len(data)

for i in range(0,nrdatasets) :
    if (isinstance(data.iloc[i]['P'],str)) : # penalties
        middle = data.iloc[i]['P'].find('-')
        end = data.iloc[i]['P'].find(' ')
        home_goals = int(data.iloc[i]['P'][0: middle: 1])
        away_goals = int(data.iloc[i]['P'][(middle+1): end: 1])
        home_wins = home_goals > away_goals
        data.at[i,"penalties"] = True
    else : # no penalties
        ft_middle = data.iloc[i]['FT'].find('-')
        ft_end = data.iloc[i]['FT'].find(' ')
        ft_home_goals = int(data.iloc[i]['FT'][0: ft_middle: 1])
        ft_away_goals = int(data.iloc[i]['FT'][(ft_middle+1): ft_end: 1])
        et_middle = data.iloc[i]['ET'].find('-')
        et_end = data.iloc[i]['ET'].find(' ')
        home_goals = int(data.iloc[i]['ET'][0: et_middle: 1])
        away_goals = int(data.iloc[i]['ET'][(et_middle+1): et_end: 1])
        home_wins = (home_goals-ft_home_goals) > (away_goals-ft_away_goals)
        data.at[i,"penalties"] = False
    data.at[i,"home_wins"] = float(home_wins)
    if (home_wins==1) :
        data.at[i,"sign"] = 1
    else :
        data.at[i,"sign"] = -1
    data.at[i,"home_goals"] = home_goals
    data.at[i,"away_goals"] = away_goals
    if (data.iloc[i]["year"] <= 1984) :
        data.at[i,"year_range"] = "1965 - 1984"
    elif ((data.iloc[i]["year"] > 1984) & (data.iloc[i]["year"] <= 2004)) :
        data.at[i,"year_range"] = "1985 - 2004"
    else :
        data.at[i,"year_range"] = "2005 - 2018"
#     if (data.iloc[i]["year"] <= 1982) :
#         data.at[i,"year_range"] = "1965 - 1982"
#     elif ((data.iloc[i]["year"] > 1982) & (data.iloc[i]["year"] <= 2000)) :
#         data.at[i,"year_range"] = "1983 - 2000"
#     else :
#         data.at[i,"year_range"] = "2001 - 2018"
```


```python
year=data.groupby(['year_range'])["home_wins"]
pal = sns.color_palette("Greens_d", 3)
rank = year.mean().argsort().argsort()

order_list = ['1965 - 1984','1985 - 2004','2005 - 2018']
# order_list = ['1965 - 1982','1983 - 2000','2001 - 2018']
# order_list = ['1965 - 1974','1975 - 1984','1985 - 1994','1995 - 2004','2005 - 2014','2014 - 2018']

sns.set(style="ticks",font_scale=1.5)

ax=sns.barplot(x="year_range", y="home_wins",data=data, ci=None,palette=np.array(pal[::-1])[rank],order=order_list)
ax.set(xlabel='Time period', ylabel='Percentage home wins\nafter regular time',ylim=(0.45, 0.70))

vals = ax.get_yticks()
ax.set_yticklabels(['{:,.0%}'.format(x) for x in vals])

plt.axhline(y=0.5, color='k')
sns.despine()

```


![png](output_3_0.png)


## Penalty kicks are completely fair

After 30 minutes of overtime, games are decided by penalty kicks. How does that affect this analysis? So far, I have only analyzed results across all games, not splitting them out by *when* the games was decided (extra time or penalty kicks). But, if we do this, something very interesting happens.

It turns out that if games finish before the 30 minutes of extra time are over, the home team has an amazing 70% to win the game! However, if the game goes to a series of penalty kicks, the home team advantage evaporates, with only 53% of a chance for either team to win. I show these percentages in the graph below. As you can see, there slight diminishing trend over time. However, there is still a huge benefit for the home team during extra time and much less during penalty kicks even in the last decade.


```python
pal = sns.color_palette("Greens_d", 3)
rank = year.mean().argsort().argsort()

order_list = ['1965 - 1984','1985 - 2004','2005 - 2018']
# order_list = ['1965 - 1982','1983 - 2000','2001 - 2018']

sns.set(style="ticks",font_scale=1.5)

ax=sns.barplot(x="year_range", y="home_wins", data=data,hue="penalties", ci=None, order=order_list)
ax.set(xlabel='Time period', ylabel='Percentage home wins\nafter regular time',ylim=(0.40, 0.75))

vals = ax.get_yticks()
ax.set_yticklabels(['{:,.0%}'.format(x) for x in vals])

legend_labels, _= ax.get_legend_handles_labels()
ax.legend(legend_labels, ['Extra time', 'PKs'],bbox_to_anchor=(1.05, 1), loc=2, borderaxespad=0.,title='Phase')

plt.axhline(y=0.5, color='k')

sns.despine()
```


![png](output_5_0.png)


Even though I am an AFC Ajax fan, still recovering from our 2019 semi-final against Tottenham, I have to conclude that, if anything, the away goal rule was not fair enough. Now that this rule has been abolished, we should see an even stronger tendency for home-playing teams to win during extra time: there is a clear home advantage and eliminating the rule benefits home teams.

However, this last analysis has provided us with a great solution for this problem! This is because all of the home team's advantage takes place during extra time. 

Therefore, UEFA should go one step further and eliminate extra time. Instead, games should go straight to penalty kicks after 2x90 minutes. In this scenario, both teams would have had a home advantage for 50% of the time, and their chances of winning would not depend on which team gets to play home last. Moreover, it would spare the players, and it would allow the UEFA to keep the away goal rule, eh, away. 

Now, of course, I am not expecting that UEFA will listen to a Dutch psychology professor in the USA, so we may have to live with the knowledge that they created an unfair system. Even so, as a psychological scientist this project has been interesting to me at many levels. The most important insight to draw from this is that decision making is hard, and that your intuition is often pretty wrong. The UEFA could really learn from this, so that in the future it will implement changes by more closely inspecting the data. But you, the reader, can also learn from this!

The lesson is that your subjective experience of the world is often inaccurate. Our minds are lazy, our decision-making systems are easily swayed by emotion, and your judgment rarely relies on an exhaustive analysis of the problem at hand. Normally, this is fine, because you simply do not have the time to think about every single decision you need to make in the world. However, as we have seen here, for important decisions, you might want to think a little harder, and perhaps even learn how to code. If you are interested in doing so, you can find the code for this project on my GitHub profile: https://github.com/wkool/cl_awaygoals.

### Acknowledgements
I thank Gerald Bauer and github.com/footballcsv for providing the bulk of the data for this project.

### Supplemental analyses
Here are the raw percentages for the analyses of win percentage of the home team for the three different time periods:


```python
year=data.groupby(['year_range'])["home_wins"]
year.mean()*100
```




    year_range
    1965 - 1984    65.517241
    1985 - 2004    64.102564
    2005 - 2018    60.000000
    Name: home_wins, dtype: float64



And here are the same values split by both year and phase of the game:


```python
yearxpen=data.groupby(['year_range','penalties'])["home_wins"]
yearxpen.mean()*100
```




    year_range   penalties
    1965 - 1984  False        73.684211
                 True         50.000000
    1985 - 2004  False        70.833333
                 True         53.333333
    2005 - 2018  False        65.217391
                 True         54.545455
    Name: home_wins, dtype: float64


