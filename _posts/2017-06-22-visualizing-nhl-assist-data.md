---
layout: post
title: "Visualizing NHL Assist Data"
description: "An interactive graphic"
date: "2017-06-22T15:27:00-05:00"
categories: [Sports]
tags: [d3, web scraping, hockey]
meta_img: "/assets/images/assist-screenshot.png"
---

<style>
#circle circle {
	fill: none;
	pointer-events: all;
}

svg.canvas {
	margin-left: -20%;
	font: 10px sans-serif;
}

.group path {
	fill-opacity: .35;
}

path.ribbon {
	stroke: #000;
	stroke-width: .25px;
	fill-opacity: 0.85;
}

#circle:hover path.fade {
	display: none;
}

text.team {
	font: 48px sans-serif;
	text-anchor: end;
	dominant-baseline: text-before-edge;
	fill: #333;
}

text.total {
	font: 36px sans-serif;
	text-anchor: end;
	dominant-baseline: text-before-edge;
	fill: #333;
}
</style>

<svg></svg>

This is another practice run with [D3.js](https://d3js.org/). With the Penguins winning their second straight Stanley Cup earlier this month, I wanted to look at some assist data from their championship season.

In the NHL, each time a goal is scored, up to two assists can be awarded to players on the goal scorer's team. If Player A passes the puck to Player B, who passes to Player C, and C scores, then A and B each get an assist. If only Player A had the puck before C scores, then A gets the only assist. It's also possible for nobody to have had the puck before C scores (usually because the opponent turned the puck over to him). In that case the goal was "unassisted", and no assists are awarded.

It is very natural to think about assists in terms of a [directed graph](https://en.wikipedia.org/wiki/Directed_graph). Each player is represented by a node, and if Player A assists on Player C's goal, an arrow is drawn from A to C. But what if we want to know *how many* assists Player A has made on goals by Player C? With a directed graph we would need multiple arrows, which will quickly become unreadable on a graph with 30 or so nodes.

To more easily show the number of assists, I decided to go with a [chord diagram](https://en.wikipedia.org/wiki/Chord_diagram). The thickness of the curved lines, called chords, between two players represents the number of assists between them. The thicker the chord, the more assists.

You might notice that the chords often have different thicknesses at each end. That's because they are *directed*. For example, Phil Kessel had 15 assists to Evgeni Malkin, but Malkin only had 7 assists to Kessel.

The players are arranged around the circle by position. Forwards are shown in yellow, defensemen in black, and goalies in blue (Matt Murray had two assists, but sadly no goals). Chords are colored according to the position of the player with more assists. For instance, Justin Schultz, a defenseman, had 7 assists to Evgeni Malkin, a forward, and Malkin had 6 to Schultz, so the chord between them is black.

Mouse over a player's name to only display assists involving that player. By hovering on the name you can see how many assists he had that season. Hovering over a chord will show you the number of assists between the two players in question.

You can see the code [here](https://bl.ocks.org/ensley/32def8d94f2b049bd583d8a6763503b7). It's also [hosted on GitHub](https://github.com/ensley/d3/tree/master/assists). I borrowed heavily from [Mike Bostock's visualization of Uber rides](https://bost.ocks.org/mike/uberdata/) to create this. I collected the data using the fantastic [MySportsFeeds](https://www.mysportsfeeds.com/) service and processed it in R.



<script src="//d3js.org/d3.v4.min.js"></script>
<script type="text/javascript">
	
	var width = 720,
	height = 720,
	outerRadius = Math.min(width, height) / 2 - 10,
	innerRadius = outerRadius - 24,
	textWidth = 300;


var layout = d3.chord()
	.padAngle(0.03)
	.sortSubgroups(d3.descending)
	.sortChords(d3.ascending);

var arc = d3.arc()
	.innerRadius(innerRadius)
	.outerRadius(outerRadius);

var ribbon = d3.ribbon()
	.radius(innerRadius);

var svg = d3.select('svg')
		.attr('class', 'canvas')
		.attr('width', width+textWidth)
		.attr('height', height)

svg.append('g')
		.attr('class', 'label')
	.append('text')
		.attr('class', 'team')
		.attr('x', width+textWidth)
		.attr('y', 0)
		.text('Pittsburgh Penguins')

svg = svg.append('g')
		.attr('id', 'circle')
		.attr('transform', 'translate('+width/2+','+height/2+')');

svg.append('circle')
	.attr('r', outerRadius);



d3.queue()
	.defer(d3.csv, '/assets/data/assists/roster.csv')
	.defer(d3.json, '/assets/data/assists/matrix.json')
	.await(ready);


function ready(error, roster, matrix) {
	if (error) throw error;

	var chords = layout(matrix);

	console.log(chords.groups);

	var groups = svg.selectAll('.group')
		.data(chords.groups)
		.enter().append('g')
		.attr('class', 'group')
		.on('mouseover', mouseover);

	groups.append('title')
		.text(function(d, i) {
			return roster[i].first + ' ' + roster[i].last + ': ' + d.value + ' assists';
		});

	var groupPaths = groups.append('path')
		.attr('id', function(d, i) { return 'group'+i; })
		.attr('class', 'player')
		.attr('d', arc)
		.style('fill', function(d, i) { console.log(roster[i].color); return roster[i].color; });

	var groupText = groups.append('text')
		.attr('x', 6)
		.attr('dx', 3)
		.attr('dy', 15);

	groupText.append('textPath')
		.attr('xlink:href', function(d, i) { return '#group' + i; })
		.text(function(d, i) { return roster[i].last; });

	groupText.filter(function(d, i) {
		return groupPaths._groups[0][i].getTotalLength() / 2 - 25 < this.getComputedTextLength();
	}).remove();

	var ribbons = svg.selectAll('.ribbon')
		.data(chords)
		.enter().append('path')
		.attr('class', 'ribbon')
		.style('fill', function(d) { return roster[d.source.index].color; })
		.attr('d', ribbon);

	ribbons.append('title')
		.text(function(d) {
			return roster[d.source.index].first + ' ' + roster[d.source.index].last
				+ ' → ' + roster[d.target.index].first + ' ' + roster[d.target.index].last
				+ ': ' + d.source.value + ' assists'
				+ '\n' + roster[d.target.index].first + ' ' + roster[d.target.index].last
				+ ' → ' + roster[d.source.index].first + ' ' + roster[d.source.index].last
				+ ': ' + d.target.value + ' assists'
		});

	d3.select('.label').append('text')
		.attr('class', 'total')
		.attr('x', width+textWidth)
		.attr('y', '50')
		.text('Total assists: '+d3.sum(matrix, function(row) {
			return d3.sum(row);
		}));

	function mouseover(d, i) {
		ribbons.classed('fade', function(p) {
			return p.source.index != i
				&& p.target.index != i;
		});
	}
}

</script>