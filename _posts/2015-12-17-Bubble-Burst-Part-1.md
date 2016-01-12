---
layout: post
published: draft
title: >
     Bubble Burst #1 - Implementation
category: Programming
excerpt: >
    Bubble Burst is an interesting and (somewhat) fun game that's available in most app stores in some form or other.  It looks something like this (this particular game is called bubble poke on android)
---

Bubble Burst is an interesting and (somewhat) fun game that's available in most app stores in some form or other.  It looks something like this (this particular game is called bubble poke on android):

![](http://i.imgur.com/XeiYoio.jpg)

The way the game works is that you tap on a group of two or more 'bubbles' and you then receive a score that is calculated by taking the number of bubbles in the group you popped multiplied by that same number minus one:

	score = bubblesInGroup * (bubblesInGroup - 1)
	
There's a clear benefit to popping as large a group as possible:	

![](http://i.imgur.com/8O4Tdce.png)


Once the game has begun there are no new bubbles that fall down or appear. A nearly completed game might look something like this:

![](http://i.imgur.com/4ceXZ5e.jpg)

Once a bubble group is 'popped', the grid adjusts itself to account for the holes left behind. Where there is a gap above a bubble, the bubbles above the gap are pushed down. If there are no more bubbles in a column then the column is removed and the column to its left is moved one place to the right. In short, bubbles are pushed down to the bottom right hand corner of the grid.

## So what?

The reason Bubble Burst is interesting is that the 'no new bubbles' feature of the game makes it deterministic, which means that you could write a program to beat it that doesn't have to deal with any new information. That's what we're gonna do.

## Implement it!

In considering possible implementations of the game, an initial approach might be to create a jagged or multidimensional array to store the bubbles. The bubble colours could be represented as an enum:

	enum Bubble {
		Cyan,
		Yellow,
		Green,
		Red,
		Blue
	}

	var bubbleGrid = new Bubble[][];

	popBubble(Bubble[][], Point point) {
		// 1. Group the bubbles
		// 2. Remove the group that point belongs to
		// 3. Reflow the grid
	}

The program will need to place the bubbles into groups which can then be used to ensure that only valid moves are made, and to calculate the scores of moves. Once that's done we can implement the 'Pop' function which removes a group of bubbles and increments the score counter. The algorithm for that should be straightforward: push the bubbles down and to the right.

Ok, sounds good so far. What are we trying to accomplish though? We aren't trying to implement Bubble Burst - we're trying to *beat* it. Each time a group of bubbles is popped, the grid is changed and an entirely new set of outcomes becomes possible. 

We need a way to undo a previous move to choose another one, or else we'll be stuck either playing a game the entire way through for each set of possible moves, or replaying the game to that point and *then* choosing an alternate group.

## Maybe we don't want to change the grid after all...

An alternative implementation could, instead of mutating the grid, take a copy of it each time and pop the bubble group in the copy. If we stored each grid as a node in a tree then we could search the tree to find the leaf with the highest total score. This sounds like progress!

How many grids are we going to be storing though? How much memory is that going to use?

## Some quick back of the envelope math...

In .NET arrays are 12 bytes each. We're storing a grid of 10x10 so we'll need ten arrays, and enums are just int's which are 4 bytes. That comes to 120 bytes for the arrays, and 400 bytes for the ints, for a total of 520 bytes per grid. 

Calculating the number of nodes at each depth is harder - we'll need to estimate. There's about 20 groups in our top level grid, which will reduce by one with each level. We'll also need to assume that the tree is uniform accross its leaves, which is quite unlikely but we can't do much better without iterating over it.

	estimatedMovesRemaining = movesAtTop - currentDepth
	nodesAtLevel = estimatedMovesRemaining * nodesAtPreviousLevel

	nodesAtLevel(2) = (20 - 2) * 20
			= 380

|Depth|# Nodes|
|-------|-------
|1		|20		
|2		|380	
|3		|6840	
|4		|116,280
|5		|1,860,480
|6		|27,907,200
|7		|390,700,800
|8		|5,079,110,400
|9		|60,949,324,800
|10		|670,442,572,800

<p />


At only 10 levels deep we have an estimated 670 billion possible grid states. Even if our estimate is out by an order of magnitude, we know that games can reach a depth of up to 20.

	670 billion * 520 bytes = ~317 TB

So now that we know we can't iterate over the entire tree, we'll need to use heuristics to trim our search space and prioritise grid positions that are more likely to yield high scores. 


## A simple heuristic

As a first step, let's take a look at a simple heuristic which trims the tree by taking the top three groups by score. In my current implementation this kind of heuristic is defined as a function that accepts a `GameMove` as a parameter and produces a list of new moves. It projects the parent node into a set of child nodes. 

{% highlight csharp %}
public class GameMove { 
    public IList<PointAndColour> Moves { get; private set; }
    public int Score { get; }
    public Grid GridState { get; }
}

Func<GameMove, IEnumerable<GameMove>> topThreeSelectionStrategy =
        parent => parent.GridState.Groups
            .OrderByDescending(group => group.Score).Take(3)
            .Select(group => parent.BurstBubble(group.Points.First()));
{% endhighlight %}

This function can then be plugged into a Tree to generate nodes. Given a root node and a child node creation strategy, we can lazily iterate over the nodes to find the best one:

{% highlight csharp %}
var topScore = 0;
foreach(var node in new LazyGeneratedTree<GameMove>(root, topThreeSelectionStrategy)) {
    if(node.Score > topScore) topScore = node.Score;
}
{% endhighlight %}

Running a breadth first search over the tree using this strategy produces the following result:

![](http://i.imgur.com/zwduoTY.png)

Further high scores were found after some time, though no higher scores were found before I stopped it at 30 minutes:

<table>
<thead>
<tr>
	<th>Score</th>
	<th>Depth</th>
	<th>Time</th>
	<th>Nodes Searched</th>
</tr>
</thead>
<tbody>
<tr>
	<td>202</td>
	<td>10</td>
	<td>1m21s</td>
	<td>40396</td>
</tr>
<tr>
	<td>204</td>
	<td>10</td>
	<td>2m3s</td>
	<td>46264</td>
</tr>
<tr>
	<td>206</td>
	<td>10</td>
	<td>2m8s</td>
	<td>46936</td>
</tr>
<tr>
	<td>208</td>
	<td>11</td>
	<td>11m47s</td>
	<td>117751</td>
</tr>
<tr>
	<td>212</td>
	<td>11</td>
	<td>11m52s</td>
	<td>118357</td>
</tr>
<tr>
	<td>218</td>
	<td>11</td>
	<td>11m57s</td>
	<td>119110</td>
</tr>
<tr>
	<td>224</td>
	<td>11</td>
	<td>14m44s</td>
	<td>138793</td>
</tr>
</tbody>
</table>
<p />

From this we can draw a couple of conclusions:

 1. Even though we've trimmed the tree, there's still too many nodes. Cutting out more nodes isn't an option, as even by taking just the top three we've cut out a large part of our search space. It's quite possible that there are pockets of high scoring nodes hidden beyond low scoring paths.
 2. There was only a couple of higher scores found per depth level. This might suggest that there could be significant gains to be made if we could prioritise more promising branches over less promising ones.
 3. A breadth first search is probably not the best way to search this tree, as we're spending a lot of time evaluating nodes that are below the high score. Prioritising more likely candidates could be a good alternative.

That's it for now! In part two of this blog series we'll take a look at some more interesting heuristics involving a priority tree search, as well as an optimisation to reduce the memory usage of the tree nodes.

Note: I'll be releasing all the code for this after part 2, but if you have ideas for heuristics using the above model, send me a Gist and I'll plug it in.


----------

<p></p>
Part Two can be found [here](http://georgekinsman.com/programming/2016/01/10/Bubble-Burst-Part-2.html)