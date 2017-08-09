---
layout: post
title: "Using optaplanner to plan water supplies"
date:   2017-08-06 00:00:00
---

# Code
The corresponding code is on [github](https://github.com/ririw/burning-water/tree/post), just run `sbt run` to run it. This blog post is based on the codebase tagged "post".

## Background

I'm heading to the burn this year. 
My job in the camp is to handle water logistics, both fresh water and grey water.
For those who don't know about the burning man, it's a party in the desert, with zero infrastructure.
That means you need to plan everything yourself, including water, both brining fresh water in, and moving dirty water out.
And of course, water is probably the most important element of our planning. 
Without it, we'll die.

So, I was able to produce a basic water plan in excel, but it was built on multiple assumptions (consumption rates, grey water production rates, etc).
I wanted to be able to quickly challeng my assumptions, and to do so, I want to be able to find good solutions quickly and easily. 
Excel wouldn't do. 

[Optaplanner](https://www.optaplanner.org/) is a constraint satisfaction solver, meaning I enter my constraints, and have it produce a working solution to them.
By encoding my water problem in it, I'm able to explore the effects of many different situations and contingencies.

I also wanted to provide a really simple introuction to Optaplanner. 
Their documentation assumes a high level of familiarly with their ecosystem, as well as assuming that you want to use DROOLs, their business rule language.
For this project however, I wanted something simpler and easier.

## Background on constraint solving

I've fallen in love with optaplanner. 
I thinks it's because it's something I can see directly improving my ability to do things: a real jump forward.
Where previously I'd solve complex planning problems with a spreadsheet, I feel like now I've got a new ability to make a computer do the work for me.
I also see it being an under appreciated area of business planning.
For the most part, complex planning solvers are embedded in specific applications, due to the difficulty of developing them.
With tools like optaplanner, I can see potential for a lot of new applications within businesses and organizations to improve things, without having to develop (or buy) a specialized tool.

Anyway, constraint solving is basically: 
 > given a set of constraints, find a way to satisfy them.
Constraint solving is NP-hard in general.

## My problem

To model my water problem, I first thought about the constraints:

 - Water is stored in 2 water barrels, and in an RV's tank, and in several smaller bottles (for redundancy and convenience)
 - Grey water is to be stored in the RV, and in one of the water tanks once it is empty.
 - Once grey water enters a container, it may not be used for fresh water
 - We may not use more water than exists in a container
 - We may not fill a container with grey water above its capacity
 - Neat solutions are preferred, so we don't need to change barrels too often.
 - We'd prefer not to use the fresh water barrels for grey water, because that means we can't use it for fresh water next year.
 - We want to allow showers in the RV.
 - RV showers must draw from the RV water supply, and feed to the RV grey water tank
 - Showers are best if they're around the middle of the burn

There's a lot here, but of course I've built them up incrementally, which is how I suggest you do it as well.

You'll also notice some of these are hard constraints ("may not", "must", etc), that must be satisfied, and soft constraints ("like", "best", etc), which do not. 
Optaplanner recommends using a `HardSoftScore` for this, and it will aim to optimize the hard score over the soft.

Optaplanner also has several important concepts that I'll weave together in my solution:
 - `PlanningEntity` represents something optaplanner will modify to find a solution.
 - `PlanningVariable`, a variable optaplanner modifies to find a solution.
 - `ValueRangeProvider`, that gives a `PlanningVariable` a set of possible values.
 - `PlanningEntityCollectionProperty`, that gives optaplanner a set of `PlanningEntity`s to work on.
 - `PlanningScore`, that tells it where to put the score for a particular solution.
 - `PlanningSolution`, representing a solution to a set of constraints.

As I was developing it, Optaplanner was pretty good about telling me what I did wrong.
If you're confused about how a problem fits together, I recommend trying something then working through the error message Optaplanner throws up.

# [The water containers](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L180-L192)

The water containers are quite straightforward.
I use a class structure to keep track of the different types of container.
Each container has:
 - A name, for printing stuff later
 - A capacity, meaning it either _contains_ that much water at the start of the burn, or is empty and _can contain_ that much grey water at the end.
 - A flag denoting whether it is for grey water.

One of the constraints above talks about transforming water barrels into grey water barrels. 
I found it easiest to design this as a constraint, rather than modeling transformable water barrels.

{% highlight scala %}
class WaterContainer(val name: String,
                     val capacity: Double,
                     var isGrey: Boolean) {
  // Optaplanner often requires a no-args constructor so it can clone and modify things.
  def this() = this(null, 0, true)
}

class WaterBarrel(val id: Integer) extends WaterContainer(s"barrel $id", 55, false)
class GreywaterBarrel(val id: Integer) extends WaterContainer(s"grey barrel $id", 55, true)
class Boxes(c: Double) extends WaterContainer("boxwater", c, false)
class RVContainer(n: String, c: Int, g: Boolean) extends WaterContainer(n, c, g)
class RVWater() extends RVContainer("rv water", 55, false)
class RVGreyWater(nTanks: Int) extends RVContainer(s"rv grey", 28*nTanks, true)
class RVBlackWater() extends RVContainer("rv black", 21, true)
{% endhighlight %} 

# [The water use day](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L194-L201)


This object represents a day of water use.
I frequently refer to this as a `grain`, which I think is a word I took from optaplanner's docs. 
 - Water consumed (given)
 - Grey water produced (given)
 - Where the water came from (set by optaplanner)
 - Where the grey water goes (set by optaplanner)
 - The number of showers (set by optaplannner) 

Here, you can see a number of the annotations needed by optaplanner

 - The `PlanningEntity` one means that this is an object that optaplanner can manipulate
 - The `PlanningVariable` ones show which variables optaplanner can alter. 
   They also tell optaplanner where to find the values that it can use for those variables.
{% highlight scala %}
@PlanningEntity
class WaterUseDay(val day: Int, val waterUse: Double, val greyWater: Double,
                  _source: WaterContainer, _dest: WaterContainer) {
  @PlanningVariable(valueRangeProviderRefs = Array("nshowers")) var showers: Integer = 0
  @PlanningVariable(valueRangeProviderRefs = Array("containers")) var dest: WaterContainer = _dest
  @PlanningVariable(valueRangeProviderRefs = Array("containers")) var source: WaterContainer = _source
  // All the things optaplanner manipulates need no-args constructors.
  def this() = this(0, 0, 0, null, null)
}
{% endhighlight %}

# [The solution object](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L207-L224)

A solution object represents an entire problem, both the containers and the days of use.
There's a lot of sillyness here due to scala/java compatibility issues. 
The scala/java boundary was a bit tricky, but I much prefer the scala solution over the java one I started on.

Here you see:
 - Range providers, so optaplanner knows where to find the containers and shower values. 
   notice the `id` field matches the ones above in the `PlanningVariable` annotations.
 - A slot for the score. 
 - And that's pretty much it.

{% highlight scala %}
@PlanningSolution
class WaterProblem(val showerTarget: Int,
                   private val _containers: List[WaterContainer],
                   private val _grains: List[WaterUseDay]) {
  def this() = this(0, null, null)

  @ValueRangeProvider(id = "containers") @ProblemFactCollectionProperty
  val containers: java.util.List[WaterContainer] = _containers.asJava
  @ValueRangeProvider(id = "nshowers")
  val showerCapacity: CountableValueRange[Integer] = { ValueRangeFactory.createIntValueRange(0, 3)}

  @PlanningEntityCollectionProperty
  val usageGrains: java.util.List[WaterUseDay] = _grains.asJava

  @PlanningScore var score: HardSoftScore = _

  // To keep the solution neat, I've put the messy print code into a separate function.
  override def toString: String = Utils.problemToString(this)
}
{% endhighlight %}


## Scoring

The scoring is where all the magic happens.
This was also the most confusing part of the project. 
The optaplanner docs assume that you're going all in, and you'll be using their DROOLs planning system. 
I'm planning on learning how to write the DROOLs rules, but for a simple problem like this, I found them to be more effort than they're probably worth.
Particularly if you're a new user, it's off putting to have to understand a new language, just to write some simple rules.

I'll only show off two of my scoring functions, there were quite a few.

### Check barrel capacity
Basically this checks the containers have space! Super duper important!
It does so by:

 1. Making a mapping, container → capacity
 2. Over each grain, subtract water and grey water use from the corresponding container.
 3. Find the RV fresh water and the RV grey water containers, because I'll need references to them to do the shower accounting in step 4.
 4. Over each grain, find showers, and appropriately handle their 3GA of water use and grey water production.

You can see it returning a `HardSoftScore`, with soft score zero and hard score set to the problem count, ie, the number of overcapacity containers.
{% highlight scala %}
def verifyCapacity(waterProblem: WaterProblem): HardSoftScore = {
    val containerCapacities = mutable.HashMap(waterProblem.containers.asScala.map(c => c -> c.capacity): _*)
    for (grain <- waterProblem.usageGrains.asScala) {
      // Handle direct source/dest
      if (grain.source != null) containerCapacities(grain.source) = containerCapacities(grain.source) - grain.waterUse
      if (grain.dest != null) containerCapacities(grain.dest) = containerCapacities(grain.dest) - grain.greyWater
    }
    val rvFresh = waterProblem.containers.asScala.find(_.isInstanceOf[RVWater]).get
    val rvGrey = waterProblem.containers.asScala.find(_.isInstanceOf[RVGreyWater]).get
    for (grain <- waterProblem.usageGrains.asScala) {
      containerCapacities(rvFresh) = containerCapacities(rvFresh) - grain.showers * 3
      containerCapacities(rvGrey) = containerCapacities(rvGrey) - grain.showers * 3
    }

    val problems = containerCapacities.values.count(_ < 0)
    HardSoftScore.valueOf(-problems, 0)
  }
{% endhighlight %}

### Encourage neat solutions
This encourages the planner to find "neat" solutions, where we don't change barrel too often. 
Whenever the water source or destination changes, it counts towards the soft score as a negative.

{% highlight scala %}
def neatness(waterProblem: WaterProblem): HardSoftScore = {
    var softScore = 0
    for ((a, b) <- waterProblem.usageGrains.asScala.zip(waterProblem.usageGrains.asScala.tail)) {
      if (a.source != b.source) softScore += 1
      if (a.dest != b.dest) softScore += 1
    }
    HardSoftScore.valueOf(0, -softScore)
  }
{% endhighlight %} 

## A tour of the rest of the code

I don't really like code heavy posts - github is a better place for that sort of thing. 
I will add a breakdown of things to look at though.

 - [solverconfig.xml](https://github.com/ririw/burning-water/blob/789520795d0f2f301fd37e5a230d5377207d0aa6/src/main/resources/waterplanner/solverconfig.xml) - this sets up the optaplanner config. 
   It's very simple, all it does is tell optplanner what classes to scan, the name of the scoring class, and the timout on the problem
 - [Main](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/Main.scala) - the program entry point.
   - [Line 9-34](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/Main.scala#L9-L36): problem setup, with various containers and day-grains, and camp sizes.
   - [Line 36-42](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/Main.scala#L37-L42): instantiating the solver, from the XML config.
   - [Line 43-45](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/Main.scala#L43-L45): running the solver
 - [Utils](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/Utils.scala) - a really really confusing printer for the solution. Probably best ignored.
 - [WaterProblem.scala](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala) - the problem itself, the interesting part.
   - [WaterSoutionScore](https://github.com/ririw/burning-water/blob/789520795d0f2f301fd37e5a230d5377207d0aa6/src/main/scala/waterplanner/WaterProblem.scala#L15) - this class is where the scoring happens, probably the most important part of the problem.
     - [forceRV](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L16-L29) - ensure that we don't carry water barrels in in the RV with EA.
     - [checkGrainExists](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L31-L51) - make sure every day of water goes somewhere
     - [verifyCapacity](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L53-L84) - make sure we don't go over the barrel's capacity
     - [neatness](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L82-L94) - encourage neat solutions
     - [ensureNoverlap](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L96-L138) - make sure that the barrels aren't used for fresh water if they're used for grey water.
     - [discourageBarrelGreyWater](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L140-L144) - while we could use both fresh water barrels for grey water, I'd prefer to keep one clean for next year. This codifies that objective.
     - [encourageShowers](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L146-L155) - this is our only "positive constraint". 
       All the rest deal with things we _don't_ want to happen, but this one basically says that showers are good and to try to fit lots in, by associating showers with positive returns.
     - [calulateScore](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L157-L166) - find the overall score by adding up all the rules.
   - [Various containers](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L180-L192), as discussed above
   - [The water use day](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L194-L201), discussed above
   - [The water problem](https://github.com/ririw/burning-water/blob/post/src/main/scala/waterplanner/WaterProblem.scala#L207-L224), discussed above

## An example solution

The solver, after solving for a while, spits out the following result:

    Greybarrels:0hard/-100soft
    Graincheck: 0hard/0soft
    Capacity:   0hard/0soft
    neatness:   0hard/-5soft
    Noverlap:   0hard/0soft
    Showers:    0hard/48soft
    forceRV:    0hard/0soft
    total:      0hard/-57soft
            showers: |   █▄ ▄ █  |
           barrel 1: |  sss      |	54.0/55.0
           barrel 2: |     sss   |	54.0/55.0
           boxwater: |ss      s  |	33.0/39.0
      grey barrel 1: |     dddddd|	54.0/55.0
      grey barrel 2: |           |	0.0/55.0
           rv black: |           |	0.0/21.0
            rv grey: |ddddd      |	52.5/56.0
           rv water: |         ss|	54.0/55.0


- `s` means "source" of water
- `d` means "destination" for grey water
- `█` denotes two showers scheduled for that day
- `▄` denotes one

You can see we've satisfied all hard objectives - that's great news!
In terms of the soft objectives, I'm also happy with how they've all been satisfied.
 - There's good balancing over the tanks,
 - The solution is as neat as I think it could be. 
   Even the split boxwater makes sense: we need to crack the fresh water barrel right away so we can use it all and turn it into a grey water barrel.
   Note that the fresh water barrel arrives with the main group on the first day, rather than going in with EA.
 - And we managed to fit in 6 showers! Great! 

This actually found a better solution than my manual attempt, by moving the box-water use to the start of the period, and thus leaving more water for showers.
I didn't notice that option when I did it in excel.


## Conclusion

Optaplanner provides an easy way to solve a large set of interesting problems.
I've aimed to show off these capabilities in a really simple example, to make it easily approachable.
Even though this is a simple example, it has real-world applicability for me, making it doubly satisfying.
I encourage you to take time to learn the sorts of things optaplanner can do, there's a good chance you'll find something cool to do with it, even if it's something as simple as planning camp water supplies.
