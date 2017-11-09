---
layout: post
title: '"How Your Favorite Baseball Team Blows Its Money": Revisiting a FiveThirtyEight
  Analysis'
date: '2017-02-22'
tags: ["baseball"]
categories: ["Sports"]
---




In 2015, FiveThirtyEight [published an article](http://fivethirtyeight.com/features/dont-be-fooled-by-baseballs-small-budget-success-stories/) taking a look at one of the most common questions in baseball: does money buy success? Most major sports leagues in America have a "salary cap," a limit on the amount of money a team can spend on player salaries each year. This is used to promote parity across all teams - otherwise, teams with deep pockets could afford to sign all the top tier players, giving them a competitive advantage year after year. Major League Baseball doesn't have a salary cap though. The Los Angeles Dodgers paid their players a total of \$265 million in 2015, while the Cleveland Indians had a payroll of only \$59 million. So in general, FiveThirtyEight asked, does more money mean more wins?

It would appeal to our affinity for underdogs if the answer was no, but unfortunately [that is not the case](http://fivethirtyeight.com/features/how-your-favorite-baseball-team-blows-its-money/). The gray curve on each of those graphs is the league-wide trend in winning percentage as a function of salary. Basically, the upward slope indicates that teams with higher payrolls tend to win more games.

It's an interesting read. You'll notice that their x-axis is "standardized salary," ranging from about -2 to +2. The authors explain that "Salary and win percentage were standardized within each season to account for the league’s financial growth and changes in league competitiveness." They calculated a z-score for every team's payroll for each year. This means that for each team, they found the mean payroll for that year, subtracted it from the team's payroll, and divided by the year's standard deviation. The result is a number indicating how many standard deviations above or below the mean that team's payroll was. I wanted to look at a slightly different method for standardizing: percentiles. A percentile is a value below which a certain percentage of values lie. For example, the Dodgers had the highest payroll in the league in 2015, putting them in the 100th percentile; every other team's payroll was below theirs. The Pirates, on the other hand, had the 20th highest payroll of the 30 teams. They were in the 34th percentile.

I found some data put together by [another blogger](http://blog.yhat.com/posts/replicating-five-thirty-eight-in-r.html) for a similar post. This data includes team-by-team payrolls and wins and losses, as well as a table mapping old teams (like the Montreal Expos) to the present-day versions (the Washington Nationals). There is also a table of official team colors that we'll use to make the final results look nice.


{% highlight r %}
df <- read_csv('mlb-standings-and-payroll.csv')
team_lookups <- read_csv('team-lookups.csv')
team_colors <- read_csv('team-colors.csv')
glimpse(df)
{% endhighlight %}



{% highlight text %}
## Observations: 1,592
## Variables: 40
## $ tm                 <chr> "ANA", "ANA", "ANA", "ANA", "ANA", "ANA", "...
## $ year               <int> 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2...
## $ attendance         <int> 1767330, 2519280, 2253123, 2066982, 2000919...
## $ attend_per_game    <int> 21553, 31102, 27816, 25518, 24703, 28464, 3...
## $ batage             <dbl> 29.7, 28.7, 28.8, 27.7, 28.0, 28.5, 29.1, 2...
## $ page               <dbl> 28.3, 29.8, 31.5, 28.9, 28.9, 30.2, 29.0, 2...
## $ bpf                <int> 102, 102, 99, 102, 101, 100, 98, 97, 100, 1...
## $ ppf                <int> 102, 102, 99, 102, 100, 99, 97, 96, 99, 100...
## $ n_hof              <int> 2, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 2...
## $ n_aallstars        <int> 1, 2, 1, 2, 2, 1, 3, 2, 1, 4, 2, 3, 6, 1, 1...
## $ n_a_ta_s           <int> 15, 16, 11, 12, 7, 11, 12, 15, 7, 14, 13, 1...
## $ est_payroll        <int> 31135472, 41791000, 55633166, 52664167, 477...
## $ time               <time> 03:06:00, 02:58:00, 02:52:00, 03:05:00, 02...
## $ managers           <chr> "Collins", "Collins", "Collins and Maddon",...
## $ X15                <int> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,...
## $ X16                <int> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,...
## $ X17                <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,...
## $ rk                 <int> 11, 12, 23, 15, 20, 4, 19, 6, 28, 2, 13, 6,...
## $ lg                 <chr> "AL", "AL", "AL", "AL", "AL", "AL", "AL", "...
## $ g                  <int> 162, 162, 162, 162, 162, 162, 162, 162, 162...
## $ w                  <int> 84, 85, 70, 82, 75, 99, 77, 92, 65, 100, 85...
## $ l                  <int> 78, 77, 92, 80, 87, 63, 85, 70, 97, 62, 77,...
## $ wins_losses        <dbl> 0.518, 0.525, 0.432, 0.506, 0.463, 0.611, 0...
## $ r                  <dbl> 5.1, 4.9, 4.4, 5.3, 4.3, 5.3, 4.5, 5.2, 4.1...
## $ ra                 <dbl> 4.9, 4.8, 5.1, 5.4, 4.5, 4.0, 4.6, 4.5, 5.0...
## $ rdiff              <dbl> 0.2, 0.0, -0.7, 0.0, -0.2, 1.3, 0.0, 0.6, -...
## $ sos                <dbl> 0.0, 0.0, -0.2, 0.1, 0.2, 0.1, 0.1, 0.1, 0....
## $ srs                <dbl> 0.2, 0.0, -0.9, 0.1, 0.0, 1.3, 0.0, 0.7, -0...
## $ pythwl             <chr> "84-78", "81-81", "70-92", "81-81", "77-85"...
## $ luck               <int> 0, 4, 0, 1, -2, -2, -3, 1, -1, -2, 0, -3, 3...
## $ home               <chr> "46-36", "42-39", "37-44", "46-35", "39-42"...
## $ road               <chr> "38-42", "43-38", "33-48", "36-45", "36-45"...
## $ exinn              <chr> "5-8", "6-6", "4-8", "9-7", "7-8", "10-5", ...
## $ `1run`             <chr> "27-25", "24-19", "23-25", "32-23", "24-24"...
## $ vrhp               <chr> "66-55", "62-59", "51-73", "61-55", "53-60"...
## $ vlhp               <chr> "18-23", "23-18", "19-19", "21-25", "22-27"...
## $ vs_teams_above_500 <chr> "23-34", "30-36", "27-45", "42-49", "37-55"...
## $ vs_teams_below_500 <chr> "61-44", "55-41", "43-47", "40-31", "38-32"...
## $ inter              <chr> "4-12", "10-6", "6-12", "12-6", "10-8", "11...
## $ last_year          <int> 1996, 1997, 1998, 1999, 2000, 2001, 2002, 2...
{% endhighlight %}



{% highlight r %}
glimpse(team_lookups)
{% endhighlight %}



{% highlight text %}
## Observations: 45
## Variables: 2
## $ historic_team <chr> "ANA", "ARI", "ATL", "BAL", "BOS", "BRO", "BSN",...
## $ modern_team   <chr> "LAA", "ARI", "ATL", "BAL", "BOS", "LAD", "ATL",...
{% endhighlight %}



{% highlight r %}
glimpse(team_colors)
{% endhighlight %}



{% highlight text %}
## Observations: 30
## Variables: 7
## $ team_name       <chr> "Arizona Diamondbacks", "Atlanta Braves", "Bal...
## $ tm              <chr> "ARI", "ATL", "BAL", "BOS", "CHC", "CHW", "CIN...
## $ team_color      <chr> "#A71930", "#B71234", "#ed4c09", "#C60C30", "#...
## $ primary_color   <chr> "#A71930", "#002F5F", "#ed4c09", "#C60C30", "#...
## $ secondary_color <chr> "#000000", "#B71234", "#000000", "#002244", "#...
## $ league          <chr> "NL", "NL", "AL", "AL", "NL", "AL", "NL", "AL"...
## $ division        <chr> "NL West", "NL East", "AL East", "AL East", "N...
{% endhighlight %}

First of all, the data only has reliable payroll information dating back to 1985. Let's filter out all teams from before 1985, drop columns we won't be using for this analysis, and join the table to the other two.


{% highlight r %}
df <- df %>% 
  filter(year >= 1985) %>% 
  select(tm, year, w, g, wins_losses, est_payroll) %>% 
  left_join(team_lookups, by = c('tm' = 'historic_team')) %>% 
  left_join(team_colors, by = c('modern_team' = 'tm'))
glimpse(df)
{% endhighlight %}



{% highlight text %}
## Observations: 858
## Variables: 13
## $ tm              <chr> "ANA", "ANA", "ANA", "ANA", "ANA", "ANA", "ANA...
## $ year            <int> 1997, 1998, 1999, 2000, 2001, 2002, 2003, 2004...
## $ w               <int> 84, 85, 70, 82, 75, 99, 77, 92, 65, 100, 85, 9...
## $ g               <int> 162, 162, 162, 162, 162, 162, 162, 162, 162, 1...
## $ wins_losses     <dbl> 0.518, 0.525, 0.432, 0.506, 0.463, 0.611, 0.47...
## $ est_payroll     <int> 31135472, 41791000, 55633166, 52664167, 477351...
## $ modern_team     <chr> "LAA", "LAA", "LAA", "LAA", "LAA", "LAA", "LAA...
## $ team_name       <chr> "Los Angeles Angels of Anaheim", "Los Angeles ...
## $ team_color      <chr> "#B71234", "#B71234", "#B71234", "#B71234", "#...
## $ primary_color   <chr> "#B71234", "#B71234", "#B71234", "#B71234", "#...
## $ secondary_color <chr> "#000000", "#000000", "#000000", "#000000", "#...
## $ league          <chr> "AL", "AL", "AL", "AL", "AL", "AL", "AL", "AL"...
## $ division        <chr> "AL West", "AL West", "AL West", "AL West", "A...
{% endhighlight %}

So far so good. Next let's calculate the payroll percentiles for each year. To do that, we can group the teams by year and then use the `dplyr` package's [`percent_rank` function](https://cran.r-project.org/web/packages/dplyr/vignettes/window-functions.html#ranking-functions).


{% highlight r %}
df <- df %>% 
  group_by(year) %>% 
  mutate(rank = percent_rank(est_payroll))
{% endhighlight %}

To make the plots, we'll need a little bit of `ggplot` magic. We will go one division at a time and create a list of plot objects, and then use the `cowplot` package to plot them all at once.


{% highlight r %}
df$division <- as.factor(df$division)
divisions <- levels(df$division)

# create the plots, one division at a time
plots <- vector('list', 6)
for(i in seq_along(divisions)) {
  df.division <- filter(df, division == divisions[i])
  plots[[i]] <-
    ggplot(df.division,
           aes(x = rank,
               y = wins_losses,
               color = team_color)) +
    geom_point(alpha = 0.75, size = 4) +
    geom_hline(yintercept = 0.5) +
    geom_vline(xintercept = 0.5) +
    stat_smooth(data = within(df, modern_team <- NULL),
                color = 'grey',
                size = 1,
                method = 'lm',
                formula = y ~ poly(x, 2),
                se=F) +
    stat_smooth(size = 2,
                method = 'lm',
                formula = y ~ poly(x, 2),
                se=F) +
    scale_color_identity() +
    scale_x_continuous(breaks = c(0, 0.5, 1),
                       limits = c(-0.1,1.1),
                       labels = c('0%','50%','100%')) +
    scale_y_continuous(breaks = seq(0.3, 0.7, 0.1),
                       limits = c(0.25, 0.75)) +
    facet_wrap(~modern_team, ncol = 5, scales = 'free_x') +
    theme_fivethirtyeight() +
    ggtitle(divisions[i])
}

cowplot::plot_grid(plotlist = plots, ncol = 1)
{% endhighlight %}

![plot of chunk unnamed-chunk-5](/assets/images/Rfig/unnamed-chunk-5-1.svg)

Compare FiveThirtyEight's results, using z-scores, to mine above. They're actually pretty similar. The plots for some of the teams got "stretched" horizontally a little, which I think makes it easier to determine trends.

Let's take this a little further. Notice the grey lines on each plot. Those are all the same - the average winning percentage at each payroll rank across the entire league. Teams would obviously prefer to be above that line, meaning that given their spending they won more games than one would expect. So it seems like if we took the distance from each point to the line and averaged them, we would have one number representing each team's "bang for their buck". Once we have those, we can put the teams in order from wisest to least-wise spenders.

Let's do it!


{% highlight r %}
fit <- lm(wins_losses ~ poly(rank, 2), data=df)

df <- df %>% 
  mutate(expected_winpct = predict(fit, newdata = data.frame(rank = rank))) %>%
  mutate(expected_w = expected_winpct*g) %>% 
  mutate(diff_w = w - expected_w)

rankings <- df %>%
  group_by(modern_team) %>%
  summarize(avg_diff = mean(diff_w)) %>%
  arrange(desc(avg_diff))

ggplot(rankings, aes(x = avg_diff, y = reorder(factor(modern_team), avg_diff))) +
  geom_segment(aes(yend = modern_team, xend = 0)) +
  geom_point() +
  scale_x_continuous(breaks = -6:6) +
  labs(x = 'Wins per year above expected, given payroll',
       y = '',
       title = '') +
  theme(panel.grid.major.y = element_blank(),
        panel.grid.minor.x = element_blank())
{% endhighlight %}

![plot of chunk unnamed-chunk-6](/assets/images/Rfig/unnamed-chunk-6-1.svg)


We can see that the *Moneyball* Oakland A's come out on top by a long shot. For a detailed year-by-year breakdown of each team from 1985 to 2015, [check out this cool app I made](http://shiny.science.psu.edu/jre206/mlb_payroll/).
