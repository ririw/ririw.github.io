---
layout: post
title: "Sintering bronze"
date:   2020-08-31 00:00:00
---

Summary
-------

This is just a quick post on how I sintered some bronze. 
This may be of interest to people who are thinking of moving into metal fabrication.
I'm interested in sintering as a way to 3d print metal.

Bronze seems like the ideal metal to start with, for a few reasons
 
 - It's not very dangerous. The FDA allows it to be [used in cosmetics](https://www.accessdata.fda.gov/scripts/cdrh/cfdocs/cfcfr/CFRSearch.cfm?fr=73.1646&SearchTerm=bronze%20powder)
 - It has a low sintering temperature, between 800°C & 950°C 
 - It's quite accessible. I bought mine from [ebay](https://www.ebay.com.au/itm/333403927012). In particular, the types sold as makeup have very small grain size (40 micron), which is easier to sinter
 - It's quite inert, so I coudl mix it with water to form a paste without causing a chemical reaction (e.g., oxidation)

Approach
--------

1. I prepared a mix of water, isopropyl alchohol and poly-vinyl alchohol. The ratio was roughly 45:45:10, respectively. The water & isopropyl alchohol were to form the bronze into a paste, while the PVA was a binder that would remain once the solvents had evaporated
2. I mixed a small amount of the liquid mixture with the bronze powder, as little as possible to form a workeable paste
3. I pushed this mixture into a small 4mm nylon standoff tube I had lying around, and used two 4mm pins to compress the mixture into a pellet. This compression was done by hand, and assuming I pushed with approx. 10kg of force, concentrated over a 4mm cylinder, it would have reached 2Mpa.
4. I let the mixture dry for 30 mins or so
5. I pushed the mixture out into a cruicble filled with previously dried sand.
6. I put the whole thing into a furnace and shot for 820°C.
7. Once the furnace reached temperature, I let it bake for 10 mins or so, then turned off the heat.
8. After an hour or so the system had cooled to ~300°C or so, and I took the item out of the crucible, and found it had sintered well, losing about 3% of its volume.

Notes
-----

### Density
Sintering outcomes are apparently driven by density; specifically:
  - The denser the input, the denser the output
  - The hotter & longer the sintering, the denser the output
  - The denser the output, the better the pysical strength.
So the more you can compress the item in step 3, the better

### Top temperature & sintering time

I found a lot of different sintering recipes on the internet. The one I used was based on a [paper](https://www.researchgate.net/figure/Bronze-sintering-profile-used-in-experiments_fig3_282835416). However, more research suggests
 
  - Bronze's melting point is very dependent on percentage of tin, so definitely work out what your bronze alloy is and infer a temperature from the phase diagram. Mine was 90/10 bronze; 10% tin.
  - The link above has a 10 minute soak time, but many sources suggest longer and hotter. Probably something to experiment with
  - See the note on density - you should probably be measuring the density of your items after sintering to validate your time & temp.

### Shapes

I made a little cylinder. It's a bit useless. The next step is to try:

 - Extrueable shapes, where I'm easily able to compress the object
 - 3d printed paste, without compression.

My main interest is in the 3d printing 😀