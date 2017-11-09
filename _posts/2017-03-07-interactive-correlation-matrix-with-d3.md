---
layout: post
title: "Interactive Correlation Matrix with D3"
description: "Here is my first attempt at making something useful: a correlation matrix where clicking on cell (i,j) draws the scatterplot between variables i and j."
categories: [code]
tags: [d3, kaggle]
date: "2017-03-07T09:58:27-05:00"
meta_img: "/assets/images/corr_screenshot.png"
---

<style>
    #canvas {
        margin-left: -20%;
    }

    .axis path,
    .axis line {
        fill: none;
        stroke: none;
        shape-rendering: crispEdges;
    }

    .axis text {
        font-family: sans-serif;
        font-size: 12px;
    }
</style>

<svg id="canvas"></svg>

Recently I decided that if I wanted to improve my data visualization skills, I should become more familiar with [D3.js](https://d3js.org). D3 is a JavaScript library that allows you to manipulate documents by associating elements with some data that you provide. It's not exactly a visualization library, like [Highcharts](http://www.highcharts.com/) or [plotly](https://plot.ly/javascript/). It doesn't have built-in chart types. Instead, it allows you to connect existing web standards like HTML, SVG, and CSS directly to your data. This makes D3 extremely powerful and flexible, but also difficult to learn.

Hearing about the steep learning curve scared me away from D3 at first, but ultimately I'm giving it a shot. After all, I'm not a total newcomer to JavaScript, and I like to think I can pick these types of things up pretty quickly, so maybe it won't be so bad.

Here is my first attempt at making something useful: a correlation matrix where clicking on cell $$(i,j)$$ draws the scatterplot between variables $$(i)$$ and $$(j)$$. I borrowed heavily from [Karl Broman's example](https://www.biostat.wisc.edu/~kbroman/D3/corr_w_scatter/). I found it very helpful to use that as a reference, since I'm not yet comfortable enough to make anything moderately involved on my own.

I got the data from a recent [Kaggle competition](https://www.kaggle.com/c/house-prices-advanced-regression-techniques) on housing prices. I did a small amount of cleaning and removed all the categorical variables for the purposes of this illustration.

The size of the circles represent the strength of the correlation between each pair of variables. Blue indicates a positive correlation, red indicates a negative correlation. Hover over a cell to see the two variables and their correlation coefficient. Click on a cell to draw the scatterplot.

You can see the code [here](https://bl.ocks.org/ensley/b289692852c62912208b4ea5b3c8ae68). It's also [hosted on GitHub](https://github.com/ensley/d3/tree/master/correlation).

<script src="//d3js.org/d3.v3.min.js"></script>
<script type='text/javascript'>

    var w = 500,
        h = 500;
    
    var margin = {top: 50, right: 20, bottom: 70, left: 20};
    var pad = 80;
    var width = 2 * w + pad;

    var svg = d3.select('#canvas')
        .attr({
            'width': width + margin.left + margin.right,
            'height': h + margin.top + margin.bottom
        })
        .append('g')
        .attr({
            'transform': 'translate(' + margin.left + ',' + margin.top + ')',
            'width': width,
            'height': h
        });

    var corrplot = svg.append('g')
        .attr({
            'id': 'corrplot'
        });

    var scatterplot = svg.append('g')
        .attr({
            'id': 'scatterplot',
            'transform': 'translate(' + (w + pad) + ',0)'
        });

    corrplot.append('text')
        .text('Correlation matrix')
        .attr({
            'class': 'plottitle',
            'x': w/2,
            'y': -margin.top/2,
            'dominant-baseline': 'middle',
            'text-anchor': 'middle'
        });

    scatterplot.append('text')
        .text('Scatter plot')
        .attr({
            'class': 'plottitle',
            'x': w/2,
            'y': -margin.top/2,
            'dominant-baseline': 'middle',
            'text-anchor': 'middle'
        });

    var corXscale = d3.scale.ordinal().rangeRoundBands([0,w]),
        corYscale = d3.scale.ordinal().rangeRoundBands([h,0]),
        corColScale = d3.scale.linear().domain([-1,0,1]).range(['crimson','white','slateblue']);
    var corRscale = d3.scale.sqrt().domain([0,1]);

    d3.json('/assets/data/d3-correlation-data/housing.json', function(err, data) {
        var nind = data.ind.length,
            nvar = data.vars.length;

        corXscale.domain(d3.range(nvar));
        corYscale.domain(d3.range(nvar));
        corRscale.range([0,0.5*corXscale.rangeBand()]);

        var corr = [];
        for (var i = 0; i < data.corr.length; ++i) {
            for (var j = 0; j < data.corr[i].length; ++j) {
                corr.push({row: i, col: j, value:data.corr[i][j]});
            }
        }

        var cells = corrplot.append('g')
            .attr('id', 'cells')
            .selectAll('empty')
            .data(corr)
            .enter().append('g')
            .attr({
                'class': 'cell'
            })
            .style('pointer-events', 'all');

        var rects = cells.append('rect')
            .attr({
                'x': function(d) { return corXscale(d.col); },
                'y': function(d) { return corXscale(d.row); },
                'width': corXscale.rangeBand(),
                'height': corYscale.rangeBand(),
                'fill': 'none',
                'stroke': 'none',
                'stroke-width': '1'
            });

        var circles = cells.append('circle')
            .attr('cx', function(d) {return corXscale(d.col) + 0.5*corXscale.rangeBand(); })
            .attr('cy', function(d) {return corXscale(d.row) + 0.5*corYscale.rangeBand(); })
            .attr('r', function(d) {return corRscale(Math.abs(d.value)); })
            .style('fill', function(d) { return corColScale(d.value); });

        corrplot.selectAll('g.cell')
            .on('mouseover', function(d) {
                d3.select(this)
                    .select('rect')
                    .attr('stroke', 'black');

                var xPos = parseFloat(d3.select(this).select('rect').attr('x'));
                var yPos = parseFloat(d3.select(this).select('rect').attr('y'));

                corrplot.append('text')
                    .attr({
                        'class': 'corrlabel',
                        'x': corXscale(d.col),
                        'y': h + margin.bottom*0.2
                    })
                    .text(data.vars[d.col])
                    .attr({
                        'dominant-baseline': 'middle',
                        'text-anchor': 'middle'
                    });

                corrplot.append('text')
                    .attr({
                        'class': 'corrlabel'
                        // 'x': -margin.left*0.1,
                        // 'y': corXscale(d.row)
                    })
                    .text(data.vars[d.row])
                    .attr({
                        'dominant-baseline': 'middle',
                        'text-anchor': 'middle',
                        'transform': 'translate(' + (-margin.left*0.1) + ',' + corXscale(d.row) + ')rotate(270)'
                    });

                corrplot.append('rect')
                    .attr({
                        'class': 'tooltip',
                        'x': xPos + 10,
                        'y': yPos - 30,
                        'width': 40,
                        'height': 20,
                        'fill': 'rgba(200, 200, 200, 0.5)',
                        'stroke': 'black'
                    });

                corrplot.append('text')
                    .attr({
                        'class': 'tooltip',
                        'x': xPos + 30,
                        'y': yPos - 15,
                        'text-anchor': 'middle',
                        'font-family': 'sans-serif',
                        'font-size': '14px',
                        'font-weight': 'bold',
                        'fill': 'black'
                    })
                    .text(d3.format('.2f')(d.value));
            })
            .on('mouseout', function(d) {
                d3.select('#corrtext').remove();
                d3.selectAll('.corrlabel').remove();
                d3.select(this)
                    .select('rect')
                    .attr('stroke', 'none');
                //Hide the tooltip
                d3.selectAll('.tooltip').remove();
            })
            .on('click', function(d) {
                drawScatter(d.col, d.row);
            });

        var drawScatter = function(col, row) {
            console.log('column ' + col + ', row ' + row);

            d3.selectAll('.points').remove();
            d3.selectAll('.axis').remove();
            d3.selectAll('.scatterlabel').remove();

            var xScale = d3.scale.linear()
                .domain(d3.extent(data.dat[col]))
                .range([0, w]);
            var yScale = d3.scale.linear()
                .domain(d3.extent(data.dat[row]))
                .range([h, 0]);

            var xAxis = d3.svg.axis()
                .scale(xScale)
                .orient('bottom')
                .ticks(5);

            var yAxis = d3.svg.axis()
                .scale(yScale)
                .orient('left');

            scatterplot.append('g')
                .attr('class', 'points')
                .selectAll('empty')
                .data(d3.range(nind))
                .enter().append('circle')
                .attr({
                    'class': 'point',
                    'cx': function(d) {
                        return xScale(data.dat[col][d]);
                    },
                    'cy': function(d) {
                        return yScale(data.dat[row][d]);
                    },
                    'r': 2,
                    'stroke': 'none',
                    'fill': 'black'
                });

            scatterplot.append('g')
                .attr('class', 'x axis')
                .attr('transform', 'translate(0,' + h + ')')
                .call(xAxis);

            scatterplot.append('g')
                .attr('class', 'y axis')
                .call(yAxis);

            scatterplot.append('text')
                .text(data.vars[col])
                .attr({
                    'class': 'scatterlabel',
                    'x': w/2,
                    'y': h + margin.bottom/2,
                    'text-anchor': 'middle',
                    'dominant-baseline': 'middle'
                });

            scatterplot.append('text')
                .text(data.vars[row])
                .attr({
                    'class': 'scatterlabel',
                    'transform': 'translate(' + (-pad/1.25) + ',' + (h/2) + ')rotate(270)',
                    'dominant-baseline': 'middle',
                    'text-anchor': 'middle'
                });

        }
    });



</script>