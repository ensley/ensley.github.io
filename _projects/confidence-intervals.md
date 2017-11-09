---
layout: page
title: "Visualizing Confidence Intervals"
header: "Visualizing Confidence Intervals"
description: "An aid for students in an undergraduate introductory statistics course. This interactive Shiny app is meant to illustrate the correct interpretation of confidence intervals."
---

[Click here to go to the applet.](http://shiny.science.psu.edu/jre206/confidence_intervals/)

[![Click here to go to the applet](/assets/images/shiny-ci-screenshot.png)](http://shiny.science.psu.edu/jre206/confidence_intervals/)

This applet is meant as an aid for students in an undergraduate introductory statistics course. Students often find it difficult to grasp the (rather unintuitive) interpretation of a confidence interval: in an infinite number of independent experiments, a 95% confidence interval would contain the true population parameter 95% of the time. The applet is an attempt to make this more apparent by drawing a large(ish) number of samples, creating confidence intervals around each sample mean, and showing what proportion of these intervals contained the population mean.

Despite a few minor bugs that are still present, this applet placed first in the Penn State statistics department's [Shiny Contest](http://stat.psu.edu/2015-shiny-contest-winners) in 2015. It was built using [R Shiny](https://shiny.rstudio.com/). The code is available [on GitHub](https://github.com/ensley/Confidence-Interval-Shiny).