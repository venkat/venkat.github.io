---
title: Liveability meter
layout: post
category: society
---

 As a liberal, brown-skinned immigrant living in the US, I feel like the places where I can comfortably live in this country  are limited. I have known this for awhile, but this was highlighted after the 2016 presidential election. I am lucky that I live in Brooklyn currently. New York City is about 45% immigrants and very socially liberal. I will live in Brooklyn as long as I can afford to, but it is a very costly place to live.

 Looking around for other places I can live with my family in the US, I have two basic criteria:

  1. I cannot live among Trump supporters.
  2. There should be a reasonably good chance that I get to interact with non-white folks where I live.

There are a lot more additional nuances to picking a place to live, but this a good start.

Thanks to OpenDataSoft's [publicly available 2016 election data](https://data.opendatasoft.com/explore/dataset/usa-2016-presidential-election-by-county%40public/), I created [this spreadsheet](https://docs.google.com/spreadsheets/d/1zg_9CnU6sjgumc52B7DRE_owmw0wLruKOhqtqACb7rU/edit?usp=sharing) that lists every county in the US with a "liveability" criterion. If more than 25% of a county's voters voted for Trump, it gets a "No" for liveability. If a county's population is more than 60 % white, it also gets a "No" for liveability. Counties where fewer than 25 % of voters voted for Trump and where fewer than 60% of the population is white get a "Yes" for liveability. Counties where fewer than 25 % of voters voted for Trump and where 60-70 % of the population is white get a "Maybe" for liveability.

Based on this categorization, I got a "Yes" for 54 counties and a "Maybe" for 5 counties. This is out of the 3113 counties in the United States that I have in the spreadsheet!

Note that there are a lot of caveats with this line of thinking:

  * Counties that appear diverse by numbers can in reality have deeply segregated neighborhoods. This applies for a lot of cities in the North East, sadly, including where I currently live.
  * I don't have the right data for Alaska and the Oglala Lakota county. Highlighted in red in the spreadsheet.
  * This is not officially sanctioned data. There are paid sources of this data which are more official and correct.
  * This is purely to satisfy my curiosity based on criteria that I care about. YMMV.

Here is the [code](https://gist.github.com/venkat/7cddbc7c792a40b256b254d1a0091dee) that I used to generate the data for the spreadsheet.

If you find this interesting, I would love to hear your thoughts. Tweet at me or email me. Thanks!
