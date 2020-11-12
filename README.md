# piecewiseLinearModel
This is a log for the development of the piece-wise linear model of our system

# pin at the top
## things for the day
- [ ] put all flags at one place and optimize them

- [ ] find the correct scenarios for the simulation

- [ ] automate the selection of files and scripts with different controllers and scenarios

- [ ] write a new post-processing script


- [x] reexamine the control requirement and find references for them

- [ ] check how good is the linearized model

- [ ] should we check some simulation with low refresh rate (run on real hardware and report the time)

- [ ] comment list: 2.1, 2.5, 2.8, 3.1

- [ ] figure position and figure caption

- [ ] justify references

## things in the dump
### implementation
-  modify *userHandInit.m* to isolate no hand situation
- unite the XY stage (merge OL and DMIMO)
- collision detection for DMIMO
- turn off gravity for DMIMO
- a more comprehensive sweep for the stable stiffness of the wall with OL
- rewrite *userHandInit.m*
- ~do collision detection with a region instead of a line (dead-zone/dead-band)~ this was implemented to some extent. (20191119)
- do documentation for all the save files and figures

### additional checks
- move configs to the top as well
- check the distribution of the disturbance level for a set of (20-batch) simulations

### future traps
- do everything in task space instead of joint space
- have different constraints on different directions
- distinguish joint torque and motor torque (the modeling of gear ratio)
- increase the dimension of the case (more complex wall)

# 20201110
## pp scripts for sliding
I should have committed this code in the beginning. 
now the development steps are lost. 
I'm not saying it's important but it might show some thought process. 

the old things I didn’t change b/c I might still use those metrics.
the new things have a separate space and naming. 
I added a few new "metrics" and noted them in 1108 mostly. 

the "plotting" side of things are mostly done. 
although they might doubt the usefulness of the plots, 
the plots show some interesting things as I said it in 1109. 

# 20201109
## fixed a bunch of things
it's not really bugs but I think my current simulation results for hand model are probably wrong. 
- for the disturbance, I chose a new delay time (0.1 s instead of 1 s). I did it with force model but I probably forgot it for hand model. this is why I need to rerun everything with hand.
- I should have used *exeScript.m* to test for all controllers under both disturbance thoroughly before moving to LS. maybe the new 0.1 s delay time was not so good. maybe 0.2 s is better. but that means the trajs need to be regenerated. 
- hand and force model both have noise but the delays are a bit different. force model is set by `pulseDelay(3)` and hand model now `hand_noise_delay`. the delay for the hand is longer b/c we also don’t want the noise when the hand force is changing. I used to overload `pulseDelay(3)` with hand's setting. I don’t really see a need for that. 
- remember there are 2 "userInit" scripts. 
- fixed some bug in *genMultiTrajs.m* a while ago. apparently I generated some trajs on angled wall and achieving the second half of the trajectory by `flipud()` was wrong. 
- *exeScript.m* now can really do a "delay" and slide. that's also how I check if 0.1 s is enough. 
- fixed hand force model in *angledWallOLwHand.slx*. now it's using the switching subsystem and the value of the disturbance is correct. I guess, still, only OL hand is affected.

## interesting finds from the trajectory stiffness plot
I'm not sure if we should report this but here's what I found. 

we did sliding with 16 trajectories. 
it's actually 8 different lines but I feel like starting from different ends could have different performance.
so it's doubled.
and I was right. 
judging from what's here, I think it can be quite simple. 
whenever the trajectory starts from the upper half of the wall (y>0), 
the performance is better, almost no penetration for the whole simulation. 
if the trajectory starts from the lower half, then there are more low points in the stiffness plot. 
there are exceptions. 
it seems that (25,25) is not so good and (-25,-25) not so bad. 
this is only for **LIMOP**. 

and then, it's **DMIMO**. 
well, it seems better than **LIMOP**. 
I cannot think of any reason. 
can it be that the gain is actually designed with a different set of QR?

# 20201108
## the winding path to trajectory stiffness plot
I've been trying a few things with this plot but so far nothing really works. 
so let me recap a little bit to help me think. 

what I did previously was no movement so there's no trajectory 
or the trajectory was just one point. 
I didn’t plot the stiffness throughout the simulation. 
instead, I computed some number/metrics and compiled all of them to report a general situation. 
I did a distribution plot based on these metrics. 

now, I have a trajectory. 
I really wanted to plot the stiffness along this trajectory 
but I cannot find a good way to do so. 
the problem is as before: 
- the penetration has some "zero-crossing" moments. 
- the negative part is ok b/c the force is also negative due to the direction. 
- the positive part means no penetration, so there shouldn't be a stiffness. 
- the exact 0 makes it infinity. 
- the near 0 makes it a very high value, some value I shouldn't claim that my device can reach. 

I dealt these things in the past. 
now that I want to draw this stiffness "trajectory" instead of compute the number, this is harder to solve. 
the first question to ask is: do I really want this plot? 
currently, my answer is still yes 
b/c it's new, it occupies space, and it might show some interesting things. 

so, I came up with new things to describe the behavior when sliding. 
- `care_ratio`: the idea is like PWM. this is the ratio of no penetration. 
- `maxOffWall`: no penetration is good but too far away from wall is bad. 
with these things, I can at least better describe the behavior. 

### how to actually do the plot
the "raw" stiffness is too much to handle. 
there are negative values, high positive values, and many ups and downs. 
I also tried to cut these unwanted values out of the data. 
that leaves the data with less points and hard to compare with each other or sum up. 
to solve the "alignment" of the data points, 
I also tried to interpolate in between to bring back the points that got taken out. 
extrapolation was also involved b/c the last part of the simulation can be taken out entirely. 
extrapolation can be replaced by assigning a constant value, like something quite high. 
currently I'm using `pchip`. `linear` will probably be fine as well. 
`spline` is trying too hard with the smoothing. 


what I really wanted for a single trajectory is a plot that sums up all 20 runs. 
but I don’t actually know if the behavior along one trajectory is similar across different runs. 
then, there is probably a "mean" plot to demonstrate the mean performance. 
there could be an "envelop" plot to show some min/max or standard deviation. 
doing this for 16 trajectories is still a bit much. 

### differentiate the plot from the table
if we want to report just a number, we could put it in a table. 
when it comes to this, I think the previous metrics are still reusable. 
and everything will be easier to sum up. 

# 20201029
## first LS run with sliding
I should say here that the simulation is very fast with sliding. 
that can mean a few things. 
- the simulation time changed from 3 s to 2 s. 
- one batch changed from probably 121 to 16 runs. 
- why can't ADAMS be this fast?

I didn’t really look into the generated data 
b/c I need new post-processing script. 

## post-processing for sliding
I probably called what I used in ToH paper "old pp". 
I probably intended to call the pp scripts with surface normal (3D) implemented new ones. 
I think now the safe bet is to write these new pp scripts for sliding based on the cosim ones. 

here, I want to list things that are different between versions to illustrate the need to have them:
- cosim uses ADAMS model. the position tracking for TCP is different from Simscape.
- hand model is different from force model b/c the signal names are different and they have different time periods to evaluate. 
- currently, angled wall is not considered in all the work pp scripts. the ones moving in that direction have "3D" in their names. they need extra checks to see the "3D" aspect is working properly. 

there are a few things that could go wrong for post-processing:
- negative penetration. in calculation, this results in a negative stiffness.
- too small penetration. this results in a very high stiffness. 

my thinking of the plot is to have the stiffness plotted along the trajectory. 
this is mostly for fanciness but it's also a good illustration. 
we have a few trajectories and they cross path with each other 
so I don’t know how that would work out. 
it's more or less the same with one plot for one trajectory 
but that would end up with 16 plots. 
of course, I can also group the 16 plots and set the x axis as time. 
then, again, it's also important to take the mean over the 20 batches. 

# 20201028
## fixed things with OL LS simulation
a bug was fixed for row vectors have to be column vectors. 
on top of that, `poseTraj` was added for OL in *runLSSliding.m*. 

I ended up doing the quick fix for the OL controller 
by bringing back the active XY stage. 
actually, an XY stage cannot take care of the angled wall, 
so the current implementation is a bush joint, 
which is very close to the slow fix I talked about. 

I don’t think it matters that much. 
it's just that with the quick fix, there will be no deviation for OL in x and y. 
that's something we cannot show that our proposed controller is superior. 
I think it's also good that we don’t advertise that 
our proposed controller has some kind of gravity compensation but OL doesn't. 
b/c that's not really an advantage but the fact that I didn’t do OL correctly. 

# 20201025
the 1021 log is still a mess. 
I thought I'd comb it to some senses but I bailed. 

it seems that *disturbInit.m* is already good for doing sliding. 
I only changed the delay before the signal to 0.1 s. 
the total simulation time would be 2.1 s. 
I thought it could be 3 s to have some consistency 
but I think 2 s could also be ok. 
for the hand force signal, it would be half the time changing and the other half holding. 
I could also do that with 3 s but currently I seem to forget how to change the params for the hand force model. 

# 20201022
## added *newHand.slx* for developing this hand motion model
hand motion model is different from the afore developed hand force model. 
ran into problems with Simulink to physical signal w/o any derivatives again. 
this is solved by setting the option for second order filtering. 
I could choose provide derivatives but Euler angle derivatives don’t really make sense. 
I'm still not sure if this is going to work. 

## misc
- added start up and shut down script for the project. now it's mostly for setting the cache folder. 
- modified *fakeDIP.m* to have more output. *exeScript.m* is also adjusted. 
- should have a separate earlier commit about adding **LIMOP** to *exeScript.m* but it's in this one commit now. 

# 20201021
it seems that there are many things waiting to be implemented for switching and sliding to work. 
to clear my head and feel less stressful, I would like to list them here. 

## a script to initialize disturbance for sliding
currently, the idea is to wait for 0.1 s for initialization stabilization. 
then, activate disturbance and the simulation is 3 s. 
with this, I might also need separate scripts for testing and batch running. 

sliding seems to work with angled wall with **DMIMO**.

## current implementation on sliding and DIP
I never wrote about what should actually happen with a real/good implementation. 
there should be a user who is holding the tool. 
the user moves the tool in the workspace, creating a motion and some collisions. 
with a functioning DIP, every time a collision happens, a pose with no penetration is computed. 
this series of poses is the input for force computing. 

in current implementation, I just realized I didn’t do many things. 
I knew I did a `fakeDIP` instead of a real one 
but I didn’t realize that, all this time, I didn’t do the user properly. 
sure, I did a hand model but that's for force not for motion. 
this is why OL is having a hot-fix with a passive XY stage. 

I should probably do a mass block attached to the world with defined motion. 
that's the hand. 
the hand and the tool are coupled by a bush joint maybe, a spring-damper system maybe. 
in this way, we might even be able to measure the force from the hand. 
this can tell us whether gravity compensation is working. 

and all the sliding and switching, I only did it for my controllers b/c they are position controllers. 
the effect of the position controller makes it seem that it's sliding. 
OL will not work b/c there is no motion. 

## what about OL
well, the first step is to make the XY stage active again. 
search for "stage" in the log for previous changes. 
then the motion would be fed through that stage. 

what about co-sim for OL?
it seems that I did a version of XY stage in ADAMS 
so this fix in Simscape should port nicely to co-sim. 
btw, while I was checking this, it seems that co-sim is not compatible with angled wall. 
that might also be something to fix. 

however, like I said, this is a "patch". 
the right way to do it is definitely doing the real hand thing I mentioned above. 
but I'm not sure how that would turn out in co-sim, or ADAMS.

another thing in mind is that there are many things I didn't do for OL. 
the OL only has force computation but no gravity compensation. 
that's a bit why there's this stage and the gravity is turned off for OL. 

one more thing is that, depending on the current implementation for the motion in ADAMS, 
angled wall might not work. 
currently, the motion is defined by a marker on the TCP and a marker on the ground. 
I'm not sure if I can but if I can make the marker on the ground turn as the wall, 
maybe this implementation could work. 
I think now I'm more towards doing it "the right way". 

but can OL and DMIMO really be done in one slx file though?
in my head, it really depends on how "hard" the spring-damper system for the hand is. 
at this point, I felt like I was thinking too much.

# 20201018
## implemented MOP controller again with real and smooth switching
previously, I did **DMOPMIMO** with "nearest" for lookup table and switching. 
of course, at the boundary, there would be discontinuity. 
this switching back and forth was so bad that I didn’t continue. 
I didn’t know how to and Lei asked me to settle with "just stick to one controller and don’t switch" for the fixed QP scenario. 
but later, they were not happy with me doing MOP but not doing a working (smooth) switching, 
so I started again. 

linear interpolation was always an option 
but I felt like it didn’t really make sense. 
I thought it would make some of the other QPs less accurate. 
another thing was that I probably didn’t find a good way to implement this. 
this was also what happened this time. 
there are a few things in simulink to do related things. 
the output of `n-D lookup table` seems to be only scalar. 
this wouldn't work for me b/c I need vector or matrix. 
this was the solution I used in **DMOPMIMO** but apparently it's not enough this time. 
there's also `direct lookup table` which can output matrix. 
however, maybe as the name "direct" implies, it didn’t seem to interpolate 
so there's no interpolated gains in between. 
it's also no good. 
I finally found a block that's just called `matrix interpolation` 
but it needed index (integer) and fraction as input. 
I didn’t like this solution as it's not clean 
but I ended up using a `1-D lookup table` to map $[-25, 25]$ to $[0,2]$. 
I used a `floor` block to separate the integer and the fraction 
and fed that to the `MI` block. 
the block works, so is the controller. 
I compared the performance moving from top right to bottom left and keeping at `(x,y) = (12.5, 12.5)`, 
which used to be a horrible QP for **DMOPMIMO**. 
and the result was in general smooth. 

the current implementation could only work for this 9-OP or actually 27-OP division of the workspace. 
breakpoints are hard coded at this time. 
the new controller is named **LIMOP** for linear interpolation. 

now that the switching is work, I need to flash out the sliding thing and maybe compare **LIMOP** against **DMIMO**. 

if this works well, I don’t really need to try the idea from yesterday. 
I think it's more or less the same. 
weighted sum is what happens in interpolation. 
I think interpolation between two points is probably enough 
and interpolation among neighbors won't change things significantly. 

## extra things about `fakeDIP`
the function is still quite simple and there are things to improve. 
I didn’t try to go for a circle or other complex path. 
this should be doable by carefully adding way points.

another thing is what Lei mentioned when the tool come at the wall with an angle. 
though I'm probably not thinking the same thing, currently the disturbance force doesn't affect the trajectory. 
this could be something to do but since it's a fake "DIP", it doesn't really matter that much. 

again, when I think about these thing, I also need to come up with research questions (RQs). 
otherwise, it's just a patch and a waste of time. 

## misc
- added **LIMOP** controller and updated reference signals in both **DMOPMIMO** and **LIMOP** to work with a trajectory instead of a fixed point

# 20201017
## new ideas on switching
the idea is basically not do switching. 
instead, do a weighted sum of things. 
it's probably like linear interpolation, which is the implementation I will go for next. 

the idea here is not "switching". 
for each computation step, compute the 9 results that are from the nearest neighbors. 
then, sum them up using a weighted sum. 
the further away, the less weight. 
in my head, it worked a lot better. 
now that I write it down, I feel like it's still switching, 
meaning there are still discontinuity moving across some boundaries. 
but it will probably be softer. 
since the weight is "continuous", maybe it's better. 

## a little problem with the delay co-sim
other things are according to plans and expectations 
but for both scenarios, the "steady-state" stiffness is higher in w/ delay than w/o. 
this I didn’t expect and I explained this to myself as a "noise" thing. 
the calculation of the stiffness is actually very unreliable. 
the numbers are quite high and it can be largely influenced by the change in small penetrations. 
this is something I should look into but currently I don’t have the time and a good solution in my head. 

## misc
- *allttest2.m* is a script that will be changed probably constantly. it's just option changes so it should be OK. so is *plotStiffLoop.m*.
- I usually make separate scripts for the plots I have in the paper so that they are fixed and the figure can be generated later with less fuss. so *plot9Box3.m* is made from *plot9Box.m* to draw the plots with delay. 
- for the delay co-sim, I only touched files related to **DMIMO** and the **OL** files were left unchanged. I didn’t run delay co-sim for OL b/c I think it's not needed. 
- the delay in DMIMO controller in *angledWallwHand.slx* is commented through at the moment. this is the only delay actually. 

# 20201005
## bugs again with co-sim post-processing
as I mentioned before, the measurements for TCP pose in Simscape and ADAMS are different. 
however, the pp scripts for co-sim were copied from those for Simscape w/o a proper checking maybe. 
on a second thought, maybe my data is fine. 
this can be a "compatibility" issue. 
on 0731, my pp scripts were right about the alignment and units. 
I probably generated the "fake" data on 0803, which is a Simscape simulation. 
so maybe that's why pp script is now last modified on 0803 and the bug is present. 

## delay against co-sim
either force or hand disturbance, the results looked similar: 
the penetration was larger, especially on the top part of the wall. 
this was something I didn't really pay attention and I couldn't make sense of it. 
I always thought that TAU did better with the upper part, making higher stiffness, 
but the penetration was actually larger? 
that didn't make sense. 

apart from that, the penetration was just large, a bit unacceptable. 
the results were for 0 to 3 steps of delay. 
I would change it to 0 to 2 steps to see if it's better. 
but this was like barely a delay. 
maybe if I count the one for breaking the algebraic loop, I could report a 1 to 3 steps of delay. 

# 20201002
## rethinking of disturbance model and potential bug
"rethinking" is probably a bit much but, 
as stated in the yesterday's log, 
disturbance model is now also a variant subsystem block, 
instead of using a switch block to switch between the 2 signals. 
this should help with the backward-compatibility. 
sometimes, I implemented something, I ran tests for it, 
then I implemented something new, I ran tests for the new thing. 
I didn't go back and check if the old thing was still working with the new thing in place. 

the potential bug, or the actual thing to remember, is that 
when I "cheat" track the position of TCP
- with Simscape, it's w.r.t the world frame, so ref point needs to be deducted before sending the TCP z back to the disturbance block
- with co-sim or ADAMS, the origin **IS** the all-zero point, so it can be fed directly to the disturbance block
it seemed that my implementations on Xinhai's PC were alright. 
so the version problem does exist. 
I think some of the codes or models on Xinhai's PC are superior. 
this is something I should solve. 
when I saw this bug on my laptop, I was shocked. 
I thought all I did in co-sim or fake must have been wrong. 
but, yeah, luckily the version on Xinhai's PC seemed right. 

## changes in files on Xinhai's PC
- *coSimADAMS.slx*: the only change seems to be the discrete time random delay, with possible steps of 0 to 3. 
- *runLargeScaleCoSim.m*: this is only changing the flag, whether it's hand or not. 
- *runManySaveCoSim.m*: this is only changing the controller type. 
- *TAU_Lib.slx*: it's updated to with the delay blocks, not the new disturbance model block.

that's mostly it. I think it's still manageable. 
I didn't make a copy of the old files before I started again a few days ago, 
which was a bit unfortunate. 
I made a backup today, which could still represent the end of ToH major revision, 
before this new minor revision. 

## missing info for doing stats after co-sim
I actually did a bunch of things after 0818 but there's no log. 
I would like to use this opportunity to document a few functions for stats. 

the script to run the simulations will probably be called something like "runLargeScale" or something. 
after this, in the result folder, there will be a number of files. 
the number should be number of batches. 
I normally run 20 batches for Simscape and 10 for co-sim.
then, "post-processing" script are used and actually "summary" scripts are used on all batches of results. 
this generates one *.mat* file that can be loaded and run *plotStiffLoop.m* to generate stiffness distribution plots. 
these are mostly what we did "before". 
no "stats" is included up until this point. 

then it should be *allttest2.m* to compare if two of those summarized results are different or not. 
this is mostly for different simulating method, for example, Simscape, co-sim, fake co-sim, and now, with delay. 
this only compares one selected metrics at a time. 
the choice can be modified in the script. 
it also saves the results of the comparison. 
as stated, this can be used on 2 summarized results for different simulating methods. 
Or it could actually take multiple pairs of 2, though you have to check if they are paired correctly. 
now it uses the suffix of the file name to separate the 2 groups. 
then the pairing within the group is done by order by names. 

the *allttest2.m* script is still a script to change. 
I probably need to tweak a few things before it could run well with the data at hands. 
as the new commit might suggest, almost everything before the for-loop is to change. 
with the results from the script, *plot9Box.m* can be used to do 9 box plot to visualize more. 
currently, it is exactly 9 box plots and they are for penetrations. 

in between, I also worked on the SOP vs. MOP paper. 
so *allttest2.m* was also used on that data with 11x11 instead of 3x3. 

# 20201001
## first attempt at sliding
currently, *fakeDIP.m* was implemented and working at a quick glance. 
I could generate a trajectory with a start and a stop pose. 
only tested with DMIMO for the moment. 
the only thing to change for other controllers should be only the ref signal. 
the ref signal was done by a "from workspace" block and the data was a matrix. 
the first column of the matrix is time and each row is a set of joint angles. 

if this is the sliding, I think the disturbance signal should change accordingly 
and maybe I should use at least a separate script for initialization of the disturbance signal. 

the current way of doing *fakeDIP.m* might be quite slow 
so it should be computed offline. 
it should be a "save and load" process in large scale simulations. 

current implementation could do both sliding (new) and point pressing (old). 
it treats point pressing as a trajectory with only one point. 
maybe b/c there is time in the trajectory, the simulation seems slower, only a little. 

## misc
- I shouldn't use switch block on an "either/or" existing part. I should use what I chose for the controller instead. in that way, the part that's not active doesn't need to have all its params. 

# 20200930
## the idea of using this controller concept
no matter what, I consider my work a different way to do force computation. 
the idea originates from the fact that
since the behavior of a position controller resembles the behavior interacting with a wall, 
a position controller can be used to do force computation in such "case" or "scenario". 
if we think along this line, we can find the next behavior to tackle, say the spring, 
and think about what controller behaves like a spring. 
currently, nothing really comes to my mind. 
(what about a position controller with a steady-state error?) 
I think I'll end up doing just a spring model. 

another thing is that I want to make an argument about stiff object interaction. 
I think it's the major interaction in (matured) haptic rendering. 
nonlinear soft object is still frontier of rendering 
and it's quite rear that you are really interacting with a linear spring. 
the graphics for deformable objects is also under development in my opinion. 
so if our approach takes care of stiff interaction, can we argue that it covers quite some ground? 

Let's draw an overall picture. 
our approach works definitely within the existing framework of haptic rendering, the rendering loop. 
it's an alternative force computation module. 
the rest, collision detection and determining of interaction point (DIP), can be left unchanged. 
either free-space or constrained, collision detection should be always working. 
when it's collision-free, DIP simply follows the trajectory of the TCP. 
(of course, here are a few problems. 
this means the result of DIP, which is a trajectory, will always be at least one step slower than the TCP. 
will this make a noticeable difference? 
this opens the opportunity to predict the next step or some other predictive things. 
if the 2 trajectory are identical, then the "error" is 0, then the controller is essentially not active. 
no, it's active. it's compensating exactly the gravity force but that creates problems as well. 
this gravity compensation part comes from the model of the device, which is a linearized model, 
meaning it's probably not accurate. 
even if it's accurate, from simulation, we know that it's "a ball sits on a pointy point". 
it's very fragile to even a small disturbance.) 
with DIP and TCP trajectory as input, the effect of the controller in free-space should be gravity compensation. 
the controller can also take care of the friction in the system as long as it's modeled properly. 
these would help with the transparency. 
when there is a collision, DIP will work based on constraints or whatever. 
here and now, we claim that we can do stiff interaction with no deformation/penetration. 

## a recap of model files
from simulation to co-sim, I redid the hand force model. 
the hand force used to be inplicit, a force computed by a block. 
the computation is based on the relative motion of the hand and the TCP and spring-damper parameters. 
these are the *disturbTest.slx* related files. 

*angledWall.slx* related files were based on these, 
with the capabilities of an angled wall as well as some other visualization assistance. 

after co-sim, hand force has to be an explicit force signal design directly by me. 
I designed the force trajectory of the hand force. 
this unifies the system in a way as now the pure force and hand force are all force signals applied to the TCP directly. 
these files are named *angledWallwHand.slx* or something similar. 
of course, co-sim files are different from these b/c the plant is from ADAMS. 
those are the *coSimADAMS.slx* related files. 
at one point during this process, I also added some more complex geometry to be in models. 
that's not really important but worth noting. 

then, there are variations in the model. 
some discrete-time delay was added to DMIMO controller. 
this is currently not our focus and should be kept comment through when running normal simulations. 
angled wall can be also considered as a variation. 
this can be achieved by changing the script. 

currently, *angledWallwHand.slx* and *coSimADAMS.slx* are under active development. 
the backward compatibility is not guaranteed but I try my best. 
I'm writing this b/c I'll change the input to the controller, the reference signal, in a big way. 
I hope everything works. 

## trajectory planner as the result of DIP
currently, my understanding is that, 
our approach cannot do sliding not b/c the approach is limited, 
but b/c we are missing DIP. 
to go around that, we assume we get perfect results from a working DIP. 
that's like a trajectory planner. 
maybe I should call this fake DIP. 
note that last time I used "fake", 
it's to see if simulation results can be treated like fake co-sim results. 
that's b/c co-sim took too long and I think the results should be more or less the same. 

# 20200927
my code is always changing. 
as it's getting bigger and bigger, it's harder to track. 
I used to track all the "configs" for different simulations 
but I guess I didn't follow through 
or that's already outdated. 
## time points in simulation
currently, I have problems with the total simulation time, 
the delay in the beginning to stable the system,
and the delay in the noise signal. 

in simulation, I used to have a 3-second simulation, 
with 1 s delay in the beginning and 2 s for the disturbance. 
moving to co-sim, I realized 1 s delay was not necessary. 
b/c the co-sim was quite slow, I shrank it down to 0.1 s. 
and, I think this would only work with pure force, 
that I also shrank the time to hold the force from 2 s to 1 s.
this was done in a commit on 20200716. 
there is also a log for that, thankfully. 
this shrinking of holding time probably wouldn't work for hand force 
b/c it takes 1 s to gradually change from 0 to the disturbance level. 
I don't think this "rising" time can be changed.

## the delay in the system
one reviewer was still quite "obsessed" with us showing some performance with delay in communication/computation. 

I started with discrete time variable time delay. 
the limitation is that the delay has to be integer times of sampling time/smallest step. 
I had a short write-up in the emails with my supervisors and I'll show it here.

### my delay write-up
**Universal Setup:** 
I directly added a delay block between the controller and the motors. QP is at the center of the workspace, origin. Single OP.

**Variant Setup:** 
the delay can be of either constant or random length. The maximal length can also be defined.

**Results:**

- Constant delay
	- With a constant delay of 1 step, the system is as before.
	- With a constant delay of 2 steps, the system is unstable, oscillating. TCP cannot hold it pose. The motors are bouncing between upper and lower saturation limits. 
	- I tried with longer constant delay. The result is similar to that of a 2-step delay.
- Random delay
	- With random delay ranging from 0 to up to 3 steps, the change in performance is unnoticeable. The errors in position in 3 axes are well within 1 mm. 
	- With random delay ranging from 0 to 4 steps, the TCP will shake from time to time. The largest error in position will exceed 1 mm. Whether the performance after this point is acceptable is subject to our “requirement”.
	- With random delay ranging from 0 to 9 steps, the TCP will still try to keep its pose and succeed from time to time, unlike the performance for a constant 2-step delay.

**Comments:**
- This is just an overall prediction. I expect the performance in co-simulation to be worse but similar. 
- This is all done by judging with my own eyes. If we really finish everything and compute the metrics, the stiffness could decrease.
- I’m not sure if the results are “significant”. The delay is dealt with in a “passive” way as we didn’t actively design something to deal with it.
- With that in mind, I still don’t know if this result is good or bad. 

### on the other hand with Lei
Lei suggested to go with a continuous time delay with length up to 1 sampling time 
and the delay should be random. 
I personally don't get why the delay should be less than the sampling time. 

### a few questions about this
when I looked at the co-sim file today, I realized I added 1-step delay to break the algebraic loop before. 
what does this mean to the whole thing?

I also don't know how to call my system. 
with delay, does that mean my system has a lower update rate? 
maybe not. b/c it's still updated at 1 kHz. 
it's just that it's always 1 step behind. 
I also did some "down-sample" tests a while back. 
how do I call that?

what's the point of testing with delay? 
if the system can handle longer delay, does that mean it's better?

## misc
- in general, I think my code is currently poorly managed. there is probably no "main version" and some of the "new features" are not updated to the previous code.

# 20200826
## studying gain scheduling
I don't know what to say but Lei suggested to address the gain scheduling (GS) issue in my MOP controller. 

so the motivation behind GS was that the control plant at different OP had different responses. 
so different controller were needed to take care of that. 
one way to check whether or not this is the case is by `bode()` plot the plant. 
I didn't really do this so I actually cannot confirm my motivation to begin with. 

# 20200818
## SOP vs. MOP statistics
I used `stiff_1stPeakVizSum` as metrics for comparison. 
out of 121 (11x11) QPs, 62 showed significant difference. 
59 didn't and 25 (5x5) out of these in the middle were supposed to do that. 
they should have no difference b/c they were the same controller. 
so out of the 96 that were supposed to have a difference, 62 did. 
out of 62 significant results, 45 had a higher mean. 
I made a Excel file to store all the numbers. 
if the result was presented like this, it's not that interesting. 
however, when only looking at the lower areas, almost all points were significantly higher. 
and if this part and the center part were taken out, whether it's higher for significant points was actually 50/50. 
of course, this kind of result was influenced by the resolution of the QPs. 
but it seemed that with more OPs, lower area on the wall was improved a lot. 
the area other than that was kinda pure luck. 
was this still b/c of saturation? 
maybe the current result is rather noisy. 
I still need to think of a way to argue about this. 
the lower area was clearly improved and that's good. 

# 20200803
I'm confused now. 
I'm not sure if this "verification" idea is good. 
## fake vs. exp
I thought there might be something wrong with cosim or whatever. 
so I would like to see if results from Simscape simulation would be "comparable". 
however, fake against exp (cosim) didn't show a better similarity. 
I really don't know what is wrong. 

# 20200802
not so good with the results
## use cosim to verify sim
I used `ttest2` to check whether the result from cosim was similar to that from sim.
the results were not so optimistic. 
only hand model with OL controller got a similar result with a 5 out of 9. 
this was something I could argue at least. 
other scenario/controller/metrics combinations were not as "good", 
showing significant difference between sim/cosim. 

I should say that maybe I shouldn't have perturb the mass distribution in ADAMS. 
I did that as a gesture to "controller takes care of model difference". 
we all agreed on this. 
however, it seemed that this one perturbation was not enough. 
to demonstrate our controller could handle modeling error, we need random perturbation maybe. 

this was something we didn't really plan well. 
now my new results were "significantly" different than my old results 
but it could be b/c of the change in the model itself. 
I should have kept everything the same when what I really wanted to do was verification. 

apart from the mass distribution, specifically, some bars were carbon fiber instead of steel, 
the hand model implementation in cosim was also different. 

(awkward sigh) OK. so I checked the definition for the Simscape model. 
it turned out: I didn't specify those carbon fiber bars are they are. 
instead, I did it like ADAMS model, defining them as steel. 
I wasn't sure if I should be happy or not when I found out about this. 
I was about to run some simulations setting them to carbon fiber so that they were the same. 
now, there's no need. 
then, I truly don't know what's giving me this significantly different results from cosim. 

## added `ttest2` scripts for statistical analysis
the idea was to, for each QP, evaluate whether the metrics concerned were similar for sim and cosim. 
*takeOutEdgePts.m* was developed to only take the 3-by-3 "edge points" in a "result dataset". 
*allttest2.m* was for generating a bunch of things that could tell us whether things were similar. 
correct files and namings were required in the folder for this script to work. 
change `theThingIdx` to get results for different metrics. 
*boxplot9.m* was developed to draw box plots given the saved results from *allttest2.m*. 
more codes were needed if this would be a figure in the paper. 

## there might be another bug
I'm breaking apart. 
the execution order of scripts was always a problem.
as my scripts got more, it's harder to keep track of them. 
*userHandInit.m* should be executed after `isWallKSet` and `isHand` were set. 
currently, this wasn't the case for *runLargeScaleCoSim.m*. 
however, if it's OL3D and pure force, *userHandInit.m* was only called once 
before `isWallKSet` was set to the correct value of 1. 
this could be the reason I wasn't satisfied with OL cosim on 20200728.

additionally, now that I added both force and hand model into 1 slx file, 
values needed to be initialized regardless of what model was active. 
so the solution to this was currently *userHandInit.m* was executed 
after controllers were loaded. 
this was implemented in *runLargeScaleCoSim.m* and *runLargeScaleFake.m*.
I checked the script. it's pretty stand-alone. 
hopefully it wouldn't affect other parts of the script.

# 20200730
## mess with pp again
the calculated metrics again had some negative values for stiffness. 
this happened with `forMeanEval` for steady force DMIMO `stiff_mDNP_steady`. 
previously, hand and force models had different ways to do this. 
this was basically whether or not we have a negative penetration. 
force model and OL were easier to have negative penetration than hand model and DMIMO, respectively. 
so in hand model pp previously, I set positive values to zero. 
I didn't do this for force model b/c it didn't need it. 
this time, I only took the mean of negative values. 
in this way, the zeros wouldn't affect the mean. 
this would probably make the penetration larger. 
see the code commit for today for details. 

this was a bit like data manipulation 
but I could also "justify" that a little. 
we would like to know the penetration when the spring was active. 
this was exactly what the current implementation was doing. 
we didn't really care about the positive values as they were in free space. 

however, this pp script would not be the same as what we used for the previous simulation results. 
this could be problematic. 

# 20200728
## problems with OL cosim
this is quite unfortunate. 
after I saw the results generated by pp scripts, there was a tendency in OL simulations 
that they tended to do better at the bottom of the workspace/wall, 
which contradicted a little with our previous findings and reasoning. 

I don't think this should happen. 
after a closer look, I think this might related to the initialization mechanism. 
for OL simulations, they were initialized by a "motion" of the TCP. 
the 6 DOFs of the TCP were all set to 0 except z direction was set to "free" 
b/c it shouldn't be constrained and would be pushed later. 
this might have a problem: when initialized the device in the lower half of the workspace/wall, 
it tended to start a few mm inside the wall. 
0.1 mm was probably fine. a few mm was quite a lot. 
b/c this was OL, there's nothing there to correct this initialization. 
force rendering was turned on only after this initialization time period passed. 

I think this might affect the performance evaluation. 
for a case like this, initialized inside the wall, there's no "free-space to constrained-space" transition. 
this transition was usually giving the max penetration. 
in other words, before the first collision happened, there's no time period that only disturbance force was active. 
this might somehow make the "first dip" smaller. 
apart from that, in the old pp scripts, I also had the concept of "steady wall". 
due to the quantization, the wall might not be 0 b/c it's simply unreachable due to the smallest step. 
this concept would be taken advantages of by the in-wall initialization, making the penetration so small. 
so currently, the metrics w/o steady seemed more favorable.

this in-wall initialization could probably be solved by using initial conditions (ICs) in ADAMS. 
I used this for DMIMO but it used to be that it would raise an error when I did it with OL. 
I thought this might be over-constraining the system so I gave up and got rid of the ICs. 
now that I saw this result and I went back to Xinhai's desktop to see if the ICs and the TCP motions were conflicting. 
it seemed working. for the time I was there. 
maybe it would still break like before as a random thing to me. 
b/c of this, I might have to rerun some (half) of the simulations. 
that's why it's quite "unfortunate".

## found a "bug" in pp scripts
I used to have 2 scripts for hand and force models separately. 
later with MOP and angled wall, I wasn't really using the hand model or the concept of steady wall, 
so this bug didn't show, probably. 
even if it did, I probably didn't notice it b/c it's quite hard to spot. 
I seldom took a look at "steady" vs. "non-steady" metrics. 

the bug was, when I calculated `stiff_mDNP_steady`, one of the metrics, 
I didn't really used the trajectory (`trajDeviatSteadyZ_mea`) after taking into account the "steady wall" effect, 
making `stiff_mDNP_steady` the same as `stiff_mDNP`, hence the figure. 
this was fixed ONLY in *ppOldCoSim.m*. 
this bug was NOT in the hand model scripts. 
I might need to process the data again to see if it's "better". 

# 20200727
"all" cosims were done, thanks to Xinhai's desktop. 
## post-processing (pp) for the major revision
it's actually been quite a while since I generated the figures for this paper. 
after MOP and angled wall, the post-processing actually changed quite a lot. 
currently, I'm looking at the results generated with the "new" pp script. 
the numbers are higher than what I reported in the paper with the Simscape simulation, 
which doesn't really make sense in my opinion. 
it could be that the "metrics" had changed quite a bit. 
so maybe I should try to analyze the data with the "old" pp script. 

I should note here that pp script for cosim should be different from that for Simscape simulation. 
the measurement from ADAMS model, especially TCP position, was in a different coordinate and unit system. 

a set of pp scripts were created b/c of this. 
they were based on "old" pp scripts, when I first processed the data for this paper. 
at least the metrics and the way to compute them was the same. 
they took the names *ppOldCoSim.m*, *ppOldHandCoSim.m*, and *sumOldCoSim.m* 
to note that they were post-processing scripts for cosim and they were based on the "old" method. 
it should be noted that I used to change "sumScript" manually to deal with hand/force situation. 
in *sumOldCoSim.m*, I made some new codes and used `isHand` as a flag to automatically switch that. 

# 20200726
## about checking the accuracy for the linearized model
I didn't say much about this. 
guess I forgot to write a log for it. 
*checkLinAccu.m* and *checkLinModAccu.slx* were used to do the checking. 
it's good that I left some comments in the code. 

# 20200725
I'm reluctant to change things these days b/c the old stuff is there and it works. 
changing means bad "compatibility". 
it's nice that the old things still work. 

## merging of force and hand model
I did the hand model w/o the Simscape spring-damper block for the co-sim. 
this opened up opportunity for having one slx file to switch between force and hand model. 

I had a bunch of flags for turning signals on and off. 
the noise could be turned off. 
the noise was also delayed for initial stabilization. 
for the hand, it's delayed longer until the hand force is not changing. 
the force feedback could also be turned off I think. 
it also had a delay in the beginning for the same reason. 
I think we could have a bunch of things renamed 
but currently, I added a new variable, `disable_FB_time`. 
this is currently only for "OL" I think. 
`coSimADAMSOL.slx` was changed accordingly 
and hopefully it's compatible with the previous things. 

## naming problem with OL/Hand
I think I had problems with naming for OL and hand model. 
they are actually orthogonal but I tend to mix them. 
for example, some of the specification for OL controller is actually in *userHandInit.m*, 
which is still needed when doing  force model OL controller for obvious reasons. 
these are things to look out for and I hope my naming of things make sense. 

# 20200722
## redo hand model as a pure force signal
previously, the hand model was done with a spring-damper block from Simscape. 
the desired disturbance force was achieved by specifying hand motion. 
then, by giving this and the TCP motion to the spring damper block, the (hand) force was generated, w/o me writing anything about the math formulation. 

now that we are running co-sim, this has to change b/c the "robot" is not in Simscape anymore. 
the hand motion was still available as it was designed by me with a "trajectory planner". 
what we needed was TCP motion we got from somewhere and the speed by differentiation. 
this was done in *redoHandModel.slx*. 
to initialize the model, I used *initCoSim.m* with the `isPureForce = 0`. 
the idea was that the force generated from my "manual" calculation should be the same as the force we got from the spring-damper block. 
the result was close, with my calculation "rising" a bit faster and earlier than the old one. 
it's also not that smooth. 
I actually like the new implementation b/c I cannot explain why there's a kind of a delay in the old one from 1 s to about 1.2 s. 

another thing I found when I took a look back at my old implementation on hand model was that I couldn't understand how random noise worked in that system. 
to the current me, the added noise never reached TCP. 
it only happened when I "collect/measure" the force. 
so was this modeling the measurement noise instead of adding a real noisy signal into the force acting upon the TCP? 
if this was the case, it made more sense. 
I had a vague memory about this discussion with Lei. 
it's probably that I had a had time trying to implement that noise and later ended up with that with that reasoning. 
however, I still don't get it why I didn't do this in the first place. 
why was I so obsessed that it should be an implicit force instead of an explicit one? 

# 20200720
when I'm knee's deep into details, I tend to forget things. 
## current status of cosim
### the story
cosim for DMIMO force model had been running for a while now and 10 batches were collected. 
I've been working on OL controller for some time now and until very recently it was still a mess. 
I tried a bunch of things to recreate the Simscape simulation. 
I did a lot of tricks in there for OL to work. 
so in ADAMS this time, I fixed all other DOFs instead of z, turned off gravity, and added more damping (a bushing) for it to have a stabler result. 
unfortunately, none of these worked. 

I then turned to the "algebraic loop" warnings b/c these warnings sometimes turned into errors for some reason. 
Lei suggested again to add a delay to break that. 
after adding that, I got a feeling that the system was "one step behind" and it's affecting the results. 

so what I got was TCP gradually moving from the wall, even from the very beginning, w/o even triggering the collision/force feedback. 
of course, these things happened quite "in parallel", not in the order I wrote them. 
b/c it's never triggering and the disturbance force didn't slow that motion from happening, I didn't know what to do. 
I thought the system might be "missing one step". 
maybe it triggered one step and it's quite large so the disturbance force couldn't counter it in a short time. 
I was really a bit destroyed by the fact that it's not working.

### the awakening
however, it turned out that I got the disturbance force in the WRONG DIRECTION! 
instead of pushing the TCP against the wall, it's pulling it away. 
no wonder there was the motion and there was nothing counteracting it. 
this was the second time I had a different direction from YZ's work and every time it's wasting quite some time. 
I didn't realize this from the DMIMO case b/c even if it's pulling, the controller still worked. 
that's probably why I needed that "fakewall" I was mentioning before. 
I didn't realize "no real penetration" was b/c I was pulling away from the wall instead of pushing. 

### the shadow
after fixing this, everything worked, kind of. 
I kept the delay to break the algebraic loop and it seemed ok. 
the "kind of" was referring to this "ADAMS initialization error" I was still getting from time to time. 
currently, to me, it seemed like an error that's randomly appearing. 
I could get it at any QP and any batch. 
this currently would just break everything and stop. 
this would be the next thing to fixed before running it large scale. 

## about hand model as disturbance
last section was getting to long, so I'd like to break it into 2 and talk about my doubts for the "future". 
b/c I thought I hit a dead-end with OL, I tried to do hand model. 
that's when I realized I did the hand model in Simscape and the way I did it could be hard to move to this cosim setup. 
for some reason, I didn't do it like the force model, having an explicit force signal disturbing the TCP's motion. 
instead, I coupled TCP's motion with another motion (hand motion) and the spring-damper in between generated the force based on these 2 motions. 
I don't remember why I chose to do it this way. 
I think there should be an important reason other than "so that I don't need to calculate the math or do differentiation to get the speed". 
I don't think I can do this in ADAMS b/c I'm not that good at it. 
I think I will return to the explicit force signal idea if I still cannot find any reason in this log that suggests not to. 

# 20200716
I didn't really write the log anymore, which was not so good. 
more logs were needed to document everything I did. 

## misc
- the hold before the disturbance was shrunk to 0.1 s to do testing. maybe it's also good to keep this to shorten the simulation time.
- post-processing was redone b/c the measurements from ADAMS model were a bit different. still ran into problems like no real penetration. made a "fakewall" to pass that. see code. *postProcFcnLS3dCoSim.m* was renamed after *postProcFcnLargeScale.m*.
- finished the whole thing with force model DMIMO all the way to post-processing.

# 20200703
had hiccups here and there and the things were finally working now.
however, with GUI co-sim ran 41.5 mins for a 3 s simulation. 
w/o GUI it still took 40 mins. 
this time could be a problem but it also made some sense. 
3 s simulation with 1 ms time step gives 3000 steps. 
if each step was completed with around 1 s, that's 3000 s, 50 min. 
if this was really what happened, there might be not so much room for shortening the time. 
on the other hand, what magic had  MATLAB done to make their simulation so fast? 
in this sense, maybe I did something terribly wrong. 

# 20200630
I was confused about moving forward as many things are happening these days.
I think I could start with Lei's new comments on things and gradually move forward. 

# 20200629
I was still busy writing the notes trying to summarize what had happened and what didn't work 
when I received a msg from YZ that he finally found the "bug" I suspected. 
when I saw the way the ADAMS model under my controller collapsed, I felt like there must be a mistake instead of 
a modeling error so huge my controller couldn't handle. 
YZ was quite confident with his modeling and I agreed. 
but now, with this "bug" fixed, my controller could control ADAMS model although the performance was not as good. 
this was normal and as expected, given that the difference between model should be quite a lot. 
however, given all the other tests, maybe the "root" part was not a problem (b/c they were close to base) and so was the steel bars that were suppose to be carbon fiber (b/c they were light even as steel). 

# 20200627
I forgot to write this log. many things had happened.
currently, we are doing co-sim again with ADAMS model. 
my memory is not so good. so I would like to note everything down. 

the master student Yang Zhang (YZ) was helpig me with all of this. 
he redefined all active joints and the torques on them. 
he felt like this was the right way to go. 
he was using 2020 version so there might be a chance that something didn't come through when I used it in 2017 version. 

## ADAMS related stuff
the motor in the ADAMS model (mass-inertia property) was "defined by the user" and it was 0.15 kg per motor. 
let's call the clutter of the transmission and motor mount "the root", as it is the starting point of each branch/chain. 
the root itself is low in mass, around 0.05 kg. 
for the Simscape model, I imported the motor and the root as a whole assembly and defined the density to be close to steel. 
this gives the thing around 0.6 kg, which is quite heavy. 
because: 1) these things are quite close to the rotation axis; 
2) these things are not so "heavy"; 
3) when these things are added in the Simscape model, the torque to compensate at the OP is not changing much; 
we could probably conclude that the "difference" in the modeling was not caused by these things. 

the other mass-related problem in the old ADAMS model I found a long time ago was the carbon fiber links were defined closed to steel, with only one as carbon fiber. 
as they are the moving parts and quite far away from rotation axes or the base, maybe they pose a larger burden to compensation torques in the joints.

## a few points Lei pointed out
during the last meeting before summer, Lei pointed out a few things in my implementation and writing. 

"gain before integration" seemed not conventional. 
at first, I thought this was a major mistake. 
later, I realized it's not a problem as long as the gain is constant, which was true for our case. 
so this was more of a formalization thing instead of a major difference thing. 
I was trashed when Lei found out about this and was not so happy about it. 
I thought all of my work was a joke. 
but OK, maybe it's not that serious. 

my controller structure was based on some good webpage. 
but apparently I didn't quite get the math behind it. 
for that I had to check the math again to make sure the formulation of the problem was correct. 
I particularly didn't get the part where reference signal was introduced. 
I had to watch out for that part.  

# 20200520
## looked into the math behind LAT
I didn't find explicit low-level math for LAT. 
only the "default option" for linearization is "block by block" according to the [`linearizeOptions`](https://se.mathworks.com/help/slcontrol/ug/linearizeoptions.html). 
the other method seemed to be "numerical perturbation". 
apparently, we didn't use that one. 

a side note was that, this step could take a option for discrete time sampling frequency. 
I didn't use this directly. 
maybe I should check the result with this directly against my result. 
they should be the same I guess. 

## found another command related to Riccati equation
one reviewer mentioned algebraic Riccati equation. 
I had no idea what it is. 
basically, it's a mid-thingy to compute the gain for the controller. 
this was taken care of by MATLAB so I didn't know. 
after looking into it, there's a command called `idare()`.
it would be interesting to see if this command gives the same result. 

# 20200430
through writing my first journal paper, I had (and I have always had) doubts about what I actually did with this method 
and how to categorize it according to the classic literatures in the field. 
so I started to read a book on haptic rendering, a topic I intentionally didn't go into b/c I thought I was doing haptic control more and I thought haptic rendering was purely rendering algorithms, something I wouldn't touch. 
so this was my reading notes as I went through the pages. 

at least for me, the study of "haptic" was always divided into 2 with one studied and the other ignored. 
general haptics was divided into force and other things. 
of course, there's probably one level above this but let's just start from here. 
I always cared about the force part, not the other. 
the practical reason was that the implementation was much easier. 
and by the end of this paragraph, it would be clear that I wasn't even caring about force.
after this division, it's either the force was perceived through a (hard/rigid) tool. 
perceived directly through skins and things was probably too hard to do so I cared about the tool part. 
then we reached the part where it's either impedance or admittance. 
we chose impedance. 
this was not only our choices but also our perception, meaning in the literatures, 
it also seemed that there's more on this very specific branch of haptics than others. 

this book also had a definition given the assumption of impedance control. 
they said that the haptic rendering problem cared about 2 tasks, 
the force between the virtual tool and the virtual environment 
and the configuration of the virtual tool. 
normally, I only cared about the interaction force 
but apparently the configuration of the tool was also important. 
in my research, I always considered collision detection done and perfect. 
however, this was not really the "definition" of haptic rendering. 
the book said, one possible definition could be 

> Given a configuration of the haptic device $H$, 
> find a configuration of the tool $T$ that minimizes an objective function $f(H−T)$, 
> subject to environment constraints. 
> Display to the user a force $F(H,T)$ dependent on the configurations of the device and the tool.

this was what we were familiar with right now. 
basically, it's the abstract version of saying 
compute the penetration or the difference between where the virtual tool was and where it should be 
and use that to compute a force. 

and then the book said 

> In impedance rendering, the device control system should provide position information 
> and implement a force control loop.

thankfully, it also said something about the counterpart

> In that case (admittance rendering), the device control should provide device forces
> and implement a position control loop.

so, with these 2 in mind, I (still) had a hard time trying to place my work among the others. 
I was definitely measuring joint angles with encoders 
but I also seemed to have a position control as a second step. 
of course, the position control was achieved with a force control. 
personally I didn't think that my control had anything to do with admittance control. 
I thought my controller might be related to constraint-based (vs. penalty-based) rendering, 
which was probably going to appear in the later sections of the book. 

## misc points from the reading
- the concept of transparency was probably borrowed from teleoperation
- [this paper](https://doi.org/10.1109/IROS.1995.525876) could be a reference to the level of human user force in haptic interactions
- this paper also said human wouldn't notice a penetration for half an inch (almost 13 mm, very hard to believe)
- god object was a compact representation of history

# 20200429
I'm just tired of doing things manually. more automation with the code. 
## added script to run 20 batches with different controllers
added *runManySave.m* to run a number of batches with the selected list of controllers. 
it's also added as a project shortcut. 
the next step would be to also automate the choice of things like angled wall. 
the key was this script and also the `clearvars` in *runLargeScale.m*. 

## also added "plot and save" script for the results from the *sumScript3d.m*
it seemed that I missed some logs. 
*sumScript3d.m* could now save itself into a *.mat* file. 
*these files* could be then handled by *plotStiffLoop.m* to plot a list of metrics and save with good file names. 

# 20200424
## implemented wall rotation along x axis as well
it would be recommended that x and y rotations should be used separately. 
z was not implemented b/c it's a z-wall to begin with and it should not be used on its own at least. 
it would come out of the "workspace" and the part of SDMOP using interpolation would not work. 

# 20200423
current git commit corresponded to "v1" results.
## choice of metrics, again
to reiterate, choosing one (basic) metric was possible 
but the performance for all controllers would not be comparable. 
so "a" metric was needed such that the performance for OL would be OK and represent our definition 
and the performance for LQR would be reasonable (not too good). 

during the writing of the journal paper, I felt like my choice of metrics was "too picky", 
in the way that I had to define it in every detail instead of a general, well-accepted thingy like RMSE. 
this time, I tried to stick to the basics and, as Lei suggested, differentiate concept and implementation. 
that's the reason behind rewriting the post-processing script. 
of course, the other main reason was that the old one wouldn't work for this new one. 

the idea behind the choice was still the same: one for the impact/worst/minimal and one for the steady.
the minimal was pretty straight forward with basically a `min()` command. 
the steady currently had 2. 
it seems that for OL, `forceOverMeanDeviat` is the way to go, giving a value around 6000. 
for LQR based approaches, `meanOfInst` seems better with no negative value. 
it should be note that, for LQR it's still quite "noisy" as the result was quite sensitive to small changes in penetration. 
it's probably OK to use both for they were 2 different implementations from the same concept. 
another thing to do would be to tune `idealWallLim` in *postProcFcnLargeScale3d.m* (**TODO**)
this value was the "cutoff" value and it would probably affect the mean value. 

## misc
- made *plotStiffLoop.m* from *plotStiff.m* b/c of too many plots now
- learned a few tricks with the "not recommended" `eval()` from MATLAB
- the current value for `idealWallLim` was `1e6`

# 20200420
CODE, CODE, CODE!
## added OL3D controller
the important part was added the "not so simple" virtual wall. 
the name was also OK b/c it really was a 3-DOF wall instead of a "6-DOF" one. 
the detailed implementation could be seen in the block.
there's a collision "pose", a measured pose, and a surface normal basically. 
the difference of the 2 poses (actually only position) was a vector 
and the dot product with the surface normal is the motion along the surface normal direction. 
we used these things to do collision detection and force computing. 
the spring was on 3 directions. 

## shouldn't use git if you cannot use it properly
I had accidentally deleted the file *disturbTest.slx* through some git actions. 
I should have make a copy before changing anything. 
the file was lost forever but I guess *angledWall.slx* should be fine. 
from now on, this *disturbTest.slx* was actually saved from *angledWall.slx*. 
I made a zip copy of the current "model" folder which had all my slx files. 
these were the files I never made copy of. 
this in itself was a big mistake. 
from now on, I should back them up every now and then. 
and STOP using confusing git commands. PLEASE. 

## misc
- put in the missing flags in *runLargeScale.m* so that it's the same as *exeScript.m*
- the flag `isWallSet` was now automatic with OL related set to 1 and others 0 (default) for both simulation scripts
- trimmed down the unused code in *postProcFcnLargeScale3D.m* (based on *postProcFcnLargeScale.m*)

# 20200415
## started to fix the OL side of things
I went back to [20191029 log](https://github.com/easyt0re/piecewiseLinearModel#20191029) and tried to remind myself of the choices I took when I built *disturbTestBaseOL.slx*

the XY stage was of course for "fixing/compensating" the things happening perpendicular to the wall direction (surface normal). 
however, why I used an active XY stage instead of static frame transformation block was still forgotten. 
the only benefit I could think of was that it could monitor some force in order to keep the TCP there but these forces were not particularly interesting to me. 
I changed it to static frame transform this time and it seemed working. 
I saved this to a different file (*angledWallOL.slx*) just to preserve the previous file although it's already been changed a bit. 
b/c there was no "version control" for the slx files, I needed to write them down myself I guess. 

noted that these "save as" actions should have some kind of backwards compatibility. 
the idea was to try my best to keep everything unchanged. 
many visual helpers were added but they shouldn't matter. 
no-angle wall should be treated as an angled wall with a 0 angle. 

## difference between *disturbTestBaseOL.slx* and *angledWallOL.slx*
- an "inital guess" (low priority position target) was added to passive joint in chain 3 (this was actually implemented in both b/c it's also a good thing for the old version and it should work)

- a frame transformation block replaced the previous active XY stage block

- the stiffness calculation in the model was cleaned b/c it's not useful anymore and it produced annoying warnings

- the visual for start point was fixed with the rotation and it's with `start_pose(5)` instead of `wallAngleY`

- an "OL3D" controller was added based on the old OL controller 

# 20200414
a happy day
## tried to clean the code
I would like to have everything "linked" and automated. 
given the controller type and other things, the file for simulation should be decided. 
this could also be helped with the flags. 
I had too many flags now. 
maybe it's a good time to put all of them together. 
this could also help with specifying each "scenario". 

# 20200408
## added "initial" values for passive joint(s) to avoid singularity
saved *disturbTest.slx* as *angledWall.slx* to further develop this case, leaving the previous model intact. 
added a few visualization to help understanding and debugging. 

long ago, I knew that there's something wrong with how our model initialized itself. 
for the model, it's easy to initialize into a singularity pose when the calculations were correct and the pose was actually singularity-free. 
I didn't find a solution back then. 
we were thinking about adding constraints to passive joints. 
but the "constraints" in joint definition was more like "constraint forces" instead of "feasible range". 
this happened again with $(x,y) = (-20,20)$ and 10 degree angle in +Y. 
I temporarily solved this by giving the passive joint in chain 3 a low priority position target. 
this wasn't something the solver had to follow strictly but I guess it offered an "initial guess" for the solver, 
giving it a higher chance to get the result we wanted. 
the value was found by trying the range of this value. 
the correct range was something from 1.0 to 1.3. 
the singularity pose was with a value of 1.5.
so I set it at 1.15 and it could correctly initialize itself now. 
this **should** be moved to other main version of the simulation file as a patch. 
actually, more passive joints should be modified in this way to better guide the solver to the correct solution. 


# 20200404
## added *postProcFcnDeviation.m* for position error
b/c I didn't want to add more things to the *postProcFcnLargeScale.m*, so I made this additional post processing script. 
this can be incorporated into the "main file" later. 

for the sum script, I modified the *sumScript.m* directly. 
instead of loading everything, I tried to load what's really needed this time to shorten the loading time. 
should be a clever thing to do in general. 

## added a script and a helper function to draw the "9 plots"
the main file was *plotSOPMOP9.m* and the function was *myPlotPos.m*. 
at first, this was done with `subplot()` but later, b/c I wanted to modify the distance between subplots, I searched for a solution and this `axes()` method showed up. 
these 2 files include functionalities like: 
how to specify figure size, figure margin on 4 sides, figure distance in between, and get rid of ticks. 
I didn't fully understand the function *myPlotPos.m* but I knew how to achieve my goals with it. 
I thought it's a nice piece of code and I wasn't using all of it. 

## misc
- added code like `pwd` to *runLargeScale.m* to have results saved to a certain folder
- added a few things in *exeScript.m* to do angled wall but didn't follow it through
- modified Q matrix in *genMultiOPs.m* to have the same "high" gain like *linScript.m*
- added a few thing to do the angled wall
- I might need to reshuffle the files a little bit now that it's a project

# 20200311
## the angled wall implementation
first thing to remember: it cannot have a large angle. 
10 degree seemed to be fine and let's start from there. 
more things needed to be tested. 

# 20200308
not too much, not too important today.
## misc
- started MATLAB project for developing MOP further on

- although the names were the same, "controller gains" in this new version were newly generated with more info (LinOP, Q matrix)

- new metrics seem to be needed

# 20200307
## fixed difference in Q matrix
there were some mismatches in the previous test. 
to recap, SDMOP didn't perform better than DMIMO. 
a closer look revealed that SDMOP didn't have the same performance when OP and QP are the same. 
a bug was found that the Q matrix used in the *genMultiOPs.m* was different from that of *linScript.m*. 
I should save more things in the saved gains. 

# 20200218
## implemented SDMOP in *runLargeScale.m*
with LQR based controllers, or in general, the performance for pure force is worse than that for hand force. 
so we chose to implement pure force first. 
with the work from yesterday, the implementation was simple. 
however, with some quick tests, SDMOP didn't seem to out perform DMIMO, even on the OPs. 

# 20200217
come back to implementation again. i like it. 
## new ideas about MOP
I implemented MOP before. it's not that successful. 
the problem was, the switching behavior on the boundary can be unstable. 
Lei said a few things about this. 

first of all, in general, it's not a good thing to switch when the controller has integral. 
integral basically means considering the past. when I do simple switch, there is no past "memory". 
integral seems to need some time to build up. 

based on this first point, here comes the second point. 
maybe we shouldn't do a lot of switching in the first place. 
instead of switching on the fly, the logic should be, whenever the collision happens, stick to the controller in that instant. 
so there is no switching. this is probably not gain scheduling anymore. 

so the real implementation would actually be still using DMIMO controller with different gains specified before the simulation starts. 
b/c we already know the starting point of the simulation, there is no need to choose the gain or switch. 

implemented in *exeScript.m* and seemed working. 

# 20191127
implementation was again in the way of writing. 
## damping in the system
at least from what I read, adding damping in the wall model, 
in other words, modeling the wall as a spring-damper system instead of just a spring, 
wouldn't help to get the system stable. 
I found $b > KT/2 + B$ in the handbook of robotics: haptics section. 
upper case is for the wall. lower case is for the device. 
and I also found $k_e < min(2b/T, 2f_c/\delta)$ in [this stiff wall paper](https://doi.org/10.1109/TOH.2019.2893926), 
which is actually referring to [A. Okamura's paper](https://doi.org/10.1109/TRO.2005.851377). 
the second one should be an extension of the first one. 
despite of their mathematical form, I personally regarded them as guideline or "rule of thumb". 

## did some reading and learned some words and knowledge
- sustained or growing oscillation
- an order of magnitude
- velocity filter: first order 30 Hz cutoff
- it is well-known that quantization can lead to limit cycles in digital control systems 
- An impedance was considered achievable if a person could not elicit visually apparent sustained oscillations from the manipulandum handle

# 20191126
## meeting notes
- add the 3 chain conceptual block about TAU figure in the modeling Simulink model part
- check all the figures made or used in the presentations. some of them are quite good
- write more about Monte-Carlo simulation: how the numbers are randomized and based on which distribution
- make sure a clear mentioning of 1-point only linearization
- add more theoretical stuff for discrete time modeling, formulation of the optimization, objective function
- add damping to make the OL look better

# 20191122
ok. I really should stop developing (implementing). 

## tests on OL pure force range of stable wall stiffness
ok. first, something I never thought about. 
the wall was at `z = 0`. 
the max penetration due to workspace depth should be 0.025 mm. 
let's just say max force would be 15 N, making the minimal (?) stiffness 600. 
b/c if I tried to render a stiffness lower than 600, with a 15 N force, it would move out of the workspace.  

~it's just low. 
300 was ok.
400 wasn't able to stabilize itself within 1 s. 
350 was barely stable at the end of 1 s. 
with a step of 100, 700 was like the 0 damping oscillation. 
800 started to diverge, amplitude of oscillation increasing. ~ 
so it turned out that those "stable" cases above were b/c I set a limit of the TCP motion. 

so it's a problem now. 
I didn't really have a stable stiffness. 
and when the limit was really the workspace boundary, the real minimal stiffness that's not going to hit the limit was actually 1000. 
so let's just say the "feasible stiffness" range should be larger than 1000. 
but in this range, I couldn't find a value that's stable. 

## modified all the post-processing (and summary) scripts with saturation flag
I suddenly felt like maybe it's helpful to take a look at joint torques and see if saturated. 
this was added in the hope that it would help me find the "right" stiffness better. 
but on a second thought, after I implemented everything, saturation could be related more to the force level rather than rendering stiffness. 
nonetheless, maybe I could process the sweep data again to see. 

## misc
- updated *TAU_Lib.slx* to have VE as one block. this was the VE block from *disturbTestBaseOLAddHand.slx*. in this way, updated the VE block in *disturbTestBaseOL.slx* with all the persistent variables and "range" wall instead of a "line" wall. 

- added `isHand` flag to better separate the code. 

- added a scope in *disturbTestBaseOL.slx* to check if the motion limits were active. 

- added the random level thing from large scale to *exeScript.m* for OL

# 20191119
## hand on OL
I probably made a few bold comments on how the hand model would make the OL better. 
it's probably better but not that significantly better. 
the stiffness of the wall was decreased from 50000 to 7000 and I still saw oscillation. 
the step was 1000. 
7000 gave an oscillation within 1 mm, which was maybe acceptable. 
everything was tested at QP(25, 25). 

## a change of noise
for the noise in the hand disturbance, I used to introduce it in the motion signal I gave it to the "hand". 
I also had a hard time trying to get the correct gain for it b/c it's not what I calculated. 
instead I tried a bunch of gains and settled with one that's ok. 
this was also probably why the noise shouldn't be implemented like this. 

ok, let's back up a little bit. 
the noise right now was done in a way that it introduced a noise in the motion signal for the hand. 
this in turn would induce a noise in the force from the spring-damper connection. 
however, b/c it's a spring-damper system, the force was related to displacement as well as velocity. 
when using white noise in position signal, I introduced sudden changes in position, inf velocity. 
this made the damping force dominant against spring force. 
when the frequency of the noise was high, this was hard to notice. 
there was noise and it was quite noisy. 
with the correct gain (this was why for previous paragraph), the range of the noise was also fine. 
but when the frequency was low, you could see that there was just a pulse in the force when velocity was inf. 
the spring force/effect was hard to observe. 
that's why I needed magic number gain for the noise b/c it's not dominated by spring force, thus couldn't be calculated. 
a side note of the reason why I tried low frequency white noise: 
I had a feeling that this frequency was messing things up a little bit, 
plus I felt like hand motion noise could be something around 2 - 10 Hz, as I mentioned somewhere before. 

lesson learned: never use a math signal (especially a discrete one) as a physical signal. 
I did it with the noise and also with the trajectory planner. 
the next was not a good move but I would use the noise from the pure force to have that effect. 
it was ok but then the noise could be a pulling force in the very beginning of the pushing period., giving TCP a positive movement. 
this was not good and it was changed that only apply the noise when holding still. 
I didn't run into this problem before b/c in pure force, when the disturbance was on, it's always with a large enough amplitude. 
it didn't have this period of gradual increasing. 

## changed VE quite a bit
changed `isCollision` flag to a `persistent`. 
changed the dead-zone/dead-band of the wall to +/- 0.1 mm 
b/c there was some positive motion with TCP I couldn't explain. 
later I discovered that although TCP z coordinate looked fine (no collision), 
the FK calculation would put TCP into the wall, hence collision. 
this demonstrated some problems with FK guessing the TCP position. 
there was also error in that. 
based on this, changed this region to +/- 0.1 mm to avoid false detection. 
maybe a little to big but it worked fine for now. 

combining with the persistent flag, it meant that 
for collision to happen, you had to press into the wall for more than 0.1 mm. 
after that, for it to come out (change the flag), you have to be at least 0.1 mm away from the wall. 
9x9 seemed fine. 
all strange positive movements were gone. 

## misc
- changed *userHandInitFcn.m* to take another input `wall_K` b/c this was often subject to changes and it would be easier to just change it in the script version. 

- directly changed the gain on the hand noise signal to be 0, just to make sure it's dead. used to be `hand_noise_gain`. 

# 20191118
happy b-day master tongtong! 
## things happened in DMIMO hand large scale
when I was running 3x3, I reported that there were some "strange jumps" in TCP z position. 
now that I ran 11x11, this was still visible for quite a portion of all QPs. 
apart from this, there were some other things happening. 
I should have some figures here but some showed that once it's pressed, it actually had a penetration and it's hard to overcome that. 
penetration like "a dip" (what I used to have) was rare. 
now it's rather flat. 
sometimes it's pushed in and stayed there. 
sometimes it's pushed in for a little while and overcame it. 
sometimes it's oscillating a little bit between these 2 levels. 

should I pay attention to this? 
honestly, Idk. 
the first thing I thought of was saturation or the effect of the high frequency noise. 
but it's most likely not the case. 
the "push in" happened mostly around 4 - 6 N, too low to saturate. 
I turned off noise to see if that helped but it's more or less the same. 
so my current guess would be quantization. 
and I should not think about it b/c it's not that much of a deviation. 

I didn't really do a constant level sweep with the hand model. 
maybe I should do that now that it's running. 
~for some reason, the simulation was longer (used to be 1.5 s, now 3 s) but the time to get all the simulation done was shorter, about 15 min for 11x11. ~
I remembered it wrong. 
the simulation was indeed longer. 
1 batch of 11x11 now ran around 3 min, making the whole set of 20 batches around 1 hour (50 min as it seemed). 
the size of the save file varied w/o noise. 
this was understandable. 
w/ noise, it's 120 M while w/o it's 70 M. 
with noise and longer time, each simulation was with 4800+ (time) points. 
it was used to be around 1500. 

## misc
- added a flag for hand noise. it's used in the step signal that's governing the noise. this was only added to *userHandInit.m* but not the function version. 

# 20191117
## implemented "trajectory planning" for the hand motion
this was the attempt to get rid of the jumps in spring force at the start and stop of the motion. 
the reason for that jump seemed to be sudden huge change in acceleration. 
I thought about doing acceleration with sine wave or saw tooth but they were a bit hard to calculate. 
I ended up using square wave or wide pulse, meaning the acceleration was always constant but there was still a jump from 0 to that level. 
I almost gave this up and got back to sine wave. 
but I noticed that if the acceleration period was long enough, the spring force was rather ok. 
with a 15 N hand force, the max spring force would be around 16 N. 
with some noise, it was still manageable. 

this was just making the simulation more realistic, the signal more physical-like. 
I could make it with a sine wave generator but it would take more time. 
for now let's keep it as it is.
the script added was *handTrajPlan.m*. 
currently it's a script but it could be integrated into *userHandInit.m* or the function version. 

## ran *runLargeScaleHand.m* again
on my laptop I only ran 3x3. 
the position plots for the 4 corners had something strange. 
their performance was not so good and they also seemed to have a problem just to keep the TCP there. 
I believed this was the case when there was no hand model. 
did I somehow introduced something bad with the hand, or is this finally the linearization effect I had been waiting? 

the post-processing was ok. 
many things didn't work anymore but this mean stiffness based on steady point seemed to be promising. 
otherwise, I thought it was still possible to find max penetration and use that. 
these seemed to be the more robust metrics regardless of the controller and the disturbance signal. 

## misc
- b/c now I had *userHandInit.m* and *userHandInitFcn.m* separately, it's important to keep them synced. 

- a flag was added in *disturb1DInit.m* just to turn of the disturbance. together with the flag that shut off noise, this guaranteed a 0 output from the pure force source. 

- now the delay and the moving time were both set to 1 s, just for things to settle better. 

# 20191116
## potential bug in running *runLargeScaleHand.m*
*runLargeScaleHand.m* and *postProcFcnLargeScaleHand.m* were added separately to do the thing with hand. 
but there might be a bug in initializing. 
influenced by the previous version of the script, only random level of hand force and adaptive noise was defined for each simulation. 
but this might not work correctly b/c hand disturbance was not governed by only 1 number (hand force). 

so the solution was to rewrite *userHandInit.m* as a function (*userHandInitFcn.m*) instead of a script. 
again, this was not used as a `SetupFcn` for `parsim()` but rather just a function used in the routine to setup `gridIn`. 
many things were manually set but it seemed working. 
this was committed as a version to see the config for this simulation. 

## misc
- the simulation was probably not fixed-step. this mattered when I took the mean. if the out-lier part was "sampled" more, it could affect the mean more. 

# 20191115
I thought I should start a section that would always be pinned at the top. 
I would use it as a to-do list for the day to motivate me and have a sense of achievement. 
should have thought of this earlier. 
it made so much sense. 

## preliminary 3x3 runs on DMIMO with hand
from the plots, it's mostly as expected. 
there was an initial wiggle when initialized. 
that's why we had a delay to separate that. 
then there's the first jump when the hand started to move. 
there's also another when it stopped moving. 
these 2 jumps were a bit annoying yet currently I had no ways to eliminate them. 
sudden changes in the force would of course accompanied by changes in position. 
it would really be better to get rid of them. 
otherwise, I needed some other ways to avoid analyzing that part. 

ok, it should be the discontinuity in velocity or acceleration 
b/c the "trajectory" I gave to the hand was "physically" impossible. 
I could do some more "design" about the trajectory but it could be an overkill. 

other than the 2 bumps in position, the performance should be pretty consistent. 
if pure force test was more about sudden changes and worst case scenarios, 
this hand case could be more about mean performance b/c it didn't change much. 
of course this could be divided into 2 periods: whether the force was changing or not. 

## misc
- cleaned and collected most of the to-do objects. put them on top. 

# 20191114
idk but maybe it's time to split things up. 
## some history and the "file system"
the *disturbTest.slx* was first developed and for all purpose really. 
it worked with *exeScript.m* and *runLargeScale.m* with a few controllers. 
but as we have 2 types of disturbance signal now, this file was actually for pure force only. 
b/c OL controller needed zero gravity, active XY stage, and prismatic guide, 
OL related files were created. 
currently, I didn't really see a way to combine OL with other controllers. 
if the XY stage could be passive, maybe it could. 
but that's just an implementation detail and not really important. 
then the last thing was to add hand model in each of these 2 files. 
this caused new files as well. 
the pure force could be easily turned off but not for the hand. 

and b/c the model file and the initializing scripts were all different, 
I might as well develop different script for large-scale simulations and post-processing. 
this could be integrated later if needed. 

## DMIMO with hand simulation
- the simulation time of *disturbTestAddHand.slx* was governed by *userHandInit.m* now. 
- similar delay was added in the front of the hand motion. 
- *disturb1DInit.m* was still used for flags and noise. pure force disturbance was turned off by setting `dLV = 0;`.
- a noise for the hand was added to the motion of the hand rather than the force directly. 

# 20191113
## meeting notes
maybe I should change the dynamic and purpose of the meeting. 
maybe I should do "report" less. 
maybe they didn't care about what I did as long as I published papers. 
Lei left earlier. maybe I should catch up with him about the decisions. 

### decisions made
- the new outline was ok. 
- talk is cheap. show me the draft. 
- don't start over, get it done. 
- more figures. 
- pure force on OL was not good but keep it for now. 
- CDMIMO should not be here (if it's time-consuming?). 
- try to turn off gravity in DMIMO. 

# 20191112
it's CRAZY and I was losing it.

## failed to use *runLargeScale.m* to run CDMIMO simulations in general
the point of CDMIMO was that there was collision detection. 
however, that was initialized by a script and based on the position of QPs.
I didn't find a good way to implement this. 
that part had to be specified for each QP. 
that's just ugly to do and not so "compatible" with my current script. 
there's a way of having a `SetupFcn` in `parsim()` but I failed to implement that. 

I could give it another go another day but right now I was running out of time. 
this would happen to any CDMIMO simulations although I was trying to do this with pure force. 
on a high level, Lei once said that it's not good that your controller should change with your hardware. 
we were talking about including encoder resolution in controller and that's bad in his opinion. 
this was probably similar. 
maybe it shouldn't be based on QPs but rather collision points. 

I would like to have this b/c I felt like this is the only thing that's different for the 2 controller and would have a large impact on the performance. 
but maybe it's not that important right now and we could do it next time. 

## misc
- directly copied CDMIMO controller from *disturbTestAddHand.slx* to *disturbTest.slx* and saved. 

- ~moved position init section to before control structure related section in *exeScript.m* (why didn't I do this earlier?). ~ 

- this couldn't be done b/c *linAtOP.m* reinitialized `init_joints_p_sim` due to naming issues. most of my code and models were copied and pasted. many things had the same name but served different purposes now. moved `start_pose` and `stopp_pose` forward instead. 

- fixed the unit problem in *collisionDetectInit.m*. I couldn't believe I still made those. 

- modified *exeScript.m* to support running it with CDMIMO. and also probably OL, not tested. 

- added more stuff to plot in *plotStiff.m* and also comments about them. this was more like a "notebook" b/c things were supposed to run in sections. more comments to come. 

- added local maxima based metrics in *postProcFcnLargeScale.m* and *sumScript.m*. 

- after adding all these metrics, the current problem was: if I would like to delete one, I had to modified a lot. 

# 20191111
single's day
## fixed the free fall in *disturbTestAddHand.slx*
after a little bit of thinking, I realized I made a mistake. 
the gravity compensation and the force rendering could be separated very easily. 
the force compensation for the linearized point was actually the gravity compensation part. 
so I moved the "Product" block upstream a little bit, before this offset compensation, so that it would always be there. 
the outside signal would only "turn off" the force rendering part. 

the logic for on/off was also evolved. 
for "z direction", in wall should turn on.
for other directions, large deviation should turn on. 
it worked. it's just that it's switching quite a lot. 

## revisited *disturbTestBaseOL.slx* ~but failed~
ok, I made another mistake. 
when I had a working *disturbTestBaseOL.slx*, I should have left it there and saved as *disturbTestBaseOLAddHand.slx* to develop hand model. 
so now, I didn't really have the old version of *disturbTestBaseOL.slx*. 
so I made 2 copies of the latest version, which was with OL and hand model, named it as the old one, and deleted the hand model. 
for some reason, it looked even shakier than before. 
maybe it's just me. I'm used to the steadier version now. 

the reason why to revive this was to have both disturbance source as 2 cases. 
but here were some of the things that might make this case not really working. 
- it's just too shaky regardless of the stiffness of the wall. 

- for pure force, I had a delay in the front and a release in the end. the delay was fine with OL but the release would just set the TCP flying out. this was not the case for DMIMO b/c it's still activated all the time. maybe after adding collision detection, this would happen to CDMIMO as well. on a second thought, this could be better when there was noise, or worse. 

nonetheless, this had to be done. 
there had to be a way to compare the 2 controllers somehow. 
added new metric to evaluate OL pure force. 
the mean deviation under the pressing period. 
but no metrics could save OL pure force. too much oscillation. 
changed the mean of the period to the mean of the period that's actually in the wall to avoid positive mean penetration.
with high stiffness setting (currently 50000), this new metric was at a level comparable with minStiff. 
with low stiffness (5000), the calculated result would be too far off (10000+). 
if 10000 or 20000 was desired, this metric would kinda follow closely. 
the max of this metric was probably around 30000. 
there seemed to be no telling which of the following would result closer to 30000: 
`4e4, 5e4, 8e4, 10e4`. 
maybe I could do a little bit of sweep here. 

in this way, I wouldn't say it's failed but it's close. 
at least, I could write about it. 
whether or not this would make sense would be another story. 

## OL pure force simulations
did both random level with new range (8 - 15) and sweep from 1 - 15. 
the wall was set to 50000. 

I didn't really look into it closely but the difference between upper and lower half of the plane was more severe. 
the bound for `imagesc()` had to change, meaning I might need to update all plots. 
I still had to find a good story behind choosing this "new" metric. 

the sweep was also straight-forward. 
the 1 - 5 vs. 8 - 15 effect was not really showing like DMIMO. 
DMIMO 8 - 15 was flat. 
OL 8 - 15 was like a continuation of 1 - 8. 
it looked like $y = 1/x$. 
another thing worth noting was the figures for each level was super symmetric. 
the left half was probably identical to the right. 
it could the result of 0 gravity. 
another way to look at it, gravity actually contributed to the asymmetry of TAU. 

## simulation with hand as source
I would like to set down a few things. 

maybe it would be good to keep the simulation time at 1.5 s. 
then maybe 0.5 s of ramp and 1 s of flat for it to stable. 
simulation time currently was set in *disturb1DInit.m*. 
it should be set in *userHandInit.m* for this. 
now that we had different slx files for pure force or hand, we could really separate them.  
the force level should be able to be randomized and the noise should be added. 
(a thought on noise was that the current white noise was with high frequency. 
it's more like a sensory noise, not a shaking hand noise. 
for that, it should be 2 - 10 Hz. 
it's not really needed or relevant but it's a thought.) 

maybe I needed to have a set of new scripts for single run, large scale, and post-processing. 

the hope was that it would not enter/penetrate the wall. 

## a cumulated list of simulation configs for various cases
in the end, more and more things should be the same. 
### OL, pure force, random level w/ noise
- model: *disturbTestBaseOL.slx*
- in *studyBaseline0.m*, comment out `dLV = 0;` 
- in *userHandInit.m*, `isWallKSet = 1;`
- in *disturb1DInit.m*, check `addNoise, isRandomLevel` 
- currently, random level wouldn't work in single run. you may need to specify `dLV` yourself. 
- single time: `studyBaseline0; userHandInit; exeScript;` 
- large scale: remember to bring the simulink library I built for OL and also *FTtransform.m*, *skewVec3.m* 

### DMIMO, hand force, random level w/ noise
- model: *disturbTestAddHand.slx*
- in *userHandInit.m*, `isWallKSet = 0;`
- in *disturb1DInit.m*, check `addNoise == 0, isRandomLevel == 1` 
- currently, random level wouldn't work in single run. 
- single time: still missing 
- large scale: use the script *runLargeScaleHand.m* 

### for single run post-processing
this section break point was now added to the script. 
these were probably also needed: `idxQP = 1; ref_pose_ini = start_pose; pulseAmp_ini=- dLV;`

## misc
- *userHandInit.m* maybe needed some rework. some of the code was needed for OL. the other part was needed for the hand model.  

- cleaned the signal logging in *disturbTestBaseOL.slx* for *runLargeScale.m*. also made changes to the script accordingly. 

- today's commit could be a reference for configs/params as to how to run OL pure force simulations. 

- added *collisionDetectInit.m* for initializing look-up tables in CDMIMO. 

# 20191110
## added *checkInvDyn.slx* to check steady state force rendering
I had doubts about the mapping between joint torque and TCP force. 
so I added this by modifying *simTAUcheckOTorq.slx* and adding force disturbance on TCP. 
it's supposed to use *linAtOP.m* with it to check the joint torques. 
the gravity was also turned off to only see the rendering force effect. 
now I knew more things about TAU. 

apparently, when rendering a z force, with 0 gravity or no gravity compensation, the points on x and y axes were quite "special". 
along x axis (horizontal), joint 6 torque should be 0 or at least close to. 
with the same force, the lower plane needed more from joint 6 than the upper plane. 
this was actually in line with the ramp effect in stiffness. 
along y axis (vertical), joint 5 torque should be 0. 
this should be expected based on the mechanism. 
so, one mystery solved and the calculation from rendered force to joint torque was considered correct now. 

this file was probably not so useful. 
currently it's only good for this. 

## current performance of OL
*userHandInit.m* was further modified. 
there were some bugs that I hadn't thought of before and they were fixed. 
most importantly, the stop position for TCP and hand. 
**with the current params (10 N pressing) and only tested at the origin**, the rendered stiffness could reach 8000 - 8500. 
above this number, the rendered stiffness from the scope started to oscillate a lot. 
it was not until 13000 that the joints started to saturate. 
12000 was still ok. 
but again, keep in mind. this had to do with QP and pressing force. 
I had some rough "stiffness calculation and scope" to monitor it. 
it's not conclusive. 
this number was found based on whether the stiffness from the scope was oscillating (too much), 
whether any joint torques were saturated, 
and whether the rendered force was stabilized at the corresponding level with the pressing force. 
other things like the real oscillation in z position were not really considered. 

ok. so, there was this case.
when everything worked, rendering force (ref) should be the same as the rendered force and the pressing force. 
everything should at a force balance. 
but this was what happened: 
with 10 N pressing, and if I desired to render a 20000 wall, 
the rendering force would actually grow larger than 10 N. 
this meant more penetration into the wall, hence larger rendering force as a reference signal. 
but, ok, this was probably the effect of a large oscillation, not really a penetration. 
and also b/c the desired stiffness was higher, magnifying the rendering force reference signal. 
but, yeah, with a 8000 wall, you got a nicer stable end position. 

b/c the "disturbance"was different now, the way to calculate "stiffness" should also be a bit different. 
it could be a stable/mean value of the period where the pushing force was not changing anymore. 
other metrics could also be kept but I had to take care of the zero-crossing behavior in position b/c it should oscillate around the wall, which was at 0. 

how to compare the 2 (OL vs. DMIMO) was still a problem. 

## added hand model and collision detection to DMIMO
saved *disturbTest.slx* as *disturbTestAddHand.slx* to develop this. 
the hand was added with no problem. 
the XY stage was set to passive. 
I thought it's ok to have both. 
added CDMIMO as the controller. 
the performance was too good when using the hand as disturbance. 
then, collision detection was also added. 
it was implemented with a 1D look-up table and "nearest" method. 
the signal was directly measured from the model as what I did with MOP. 
this could be replaced by what's used in OL (ForKin). 

after adding collision detection, the performance dropped a little. 
more importantly, b/c at one point, all force would be deactivated, TCP would free fall a little until it's reactivated again. 
then my question was, would it be possible to separate the two? 
obviously, one was gravity compensation while the other force rendering. 
the latter should be turned off while the prior should be kept. 
if possible, this would be a real controller for all workspace for all time. 

## misc
- the simulation time was increase to 2 s for the system to stabilize. 

- there were so many things implemented with problems. it's not well-parameterized. more work needed to be done for more complex cases. 

# 20191109
## failed to recreate things found in [1106 log](https://github.com/easyt0re/piecewiseLinearModel#20191106) on OL
in misc, I logged that torque from joint 6 was perfectly 0. 
this couldn't be recreated but the torque for 5 and 6 were very close to 0 while others were bouncing between saturation limits. 
I suspected there was something wrong with the jacobian calculation but it's hard to check. 
later, I seemed to have recreated the perfect 0. 
it was still there. 
previously, I did something wrong with the delay. 
I would write this in another point. 

I also reported that the max stiffness rendered was around 3000. 
this couldn't be recreated exactly but the max was now a number below 3000 and it's actually changing. 
the changing was probably due to my QP was changing. 
then, ok, this was probably acceptable. 
and "within" the able-to-render range, `minStiff` would only be about half the desired stiffness. 
of course, this number had a very loose relationship with the real rendered stiffness. 

## added user-hand model to the OL simulation model
it did dramatically improve the stability of the rendering, almost no vibration. 
changed a few things in *userHandInit.m* to get it to work. 

I should have some description or figure to illustrate a little bit about how this hand model works. 
the overall idea was not to define a pure motion or force on the TCP. 
when no resistance, not hitting the wall, the hand and the TCP should move together, governed by the defined motion. 
when hitting the wall, the TCP would stop while the hand would still move. 
this sounded weird but this mismatch in motion created the force that the hand pushed onto the TCP. 
in this way, the defined motion was turned into a force signal. 
from now on, it should be the same as what I did previously, with a pure force signal. 
the motion signal would first increase like a ramp and then stay flat. 
consequently, the force signal would also have the same profile. 

the params from the script were pretty straight-forward. 
basically the hand was a cube mass and TCP was another. 
they were connected by a spring and damper. 
it should start by defining `hand_force`. 
this should be the same as disturbance level. 
divided by `hand_K`, this would be the amount of spring pressed when the hand would not move anymore. 
after pressing, the spring should have a positive length. 
I didn't really try the negative length thing. 
I guessed the rest was self-explanatory. 

`dLV = 0;` was added to *studyBaseline0.m* to turn off disturbance signal. 

## ran 20 batches with new range 8 - 15 N
as expected, everything seemed ok. 
there was no extreme high point and the std should be down a lot. 

## the idea of simulation
I used to think that I should run 1 simulation on both controllers. 
now that I had a pure force disturbance source and a hand model, I could have both instead of keeping only one. 
the pure force probably represented sudden changes in the force. 
the hand model probably represented gradual changes, maybe closer to real scenario. 

then maybe, another thing to do was to also have noise and randomize capability in hand model. 

## misc
- the code was at a point where it could do many different things with different "configurations". I should note all of them down maybe before I found a more elegant way to do this. I started to have wrong simulations b/c I forgot to change 1 param or 2. 

- the way to randomize level and get adaptive noise was changed in *runLargeScale.m*. 

# 20191108
## the reason behind all of this not-so-useful work
let's recap a little bit. 
I was lost again. 
this happened way too often than it should. 
the reason why I did this constant level sweep was b/c 
1) I had a feeling that under different level, the behavior of the data could be different, specifically, flat or ramp. 
2) I felt like the mean of stiffness was a weighted sum of these "figures/numbers" and the weight was the distribution of the disturbance level over all batches. 

from the sweep figures directly, the 1st point should be false. 
but there was still a chance that I did something wrong. 
maybe I should check further or do another round with higher resolution. 

~the 2nd point was yet to be checked. 
but I would like to see a stiffness over disturbance level plot, just to see if there was some correlation. ~ 
I plotted 121 QPs stiffness over disturbance level in one plot. 
I should learn by now that everything about this project is not "simple". 
there's not going to be a simple pattern/behavior across all points. 
it seemed that with level 8 - 15, the stiffness would be varying in a not-too-wide range, `[1.5, 4]`.
you could push 8 to 5 and that's what I got from previous simulations (now 100 batches). 
but below 5 was too much chaos in my opinion. 
within 8 - 15, there were still "trends". 
if everything was around a mean value, that would be great. 
but for some QPs, stiffness would increase/decrease with a increasing disturbance level. 
I guesses that's where the std came in. 

## a potential bug with `isRandomLevel == 0`
the bug would appear when constant level and with noise. 
it's not really a "bug" but the noise would not be "adaptive" to the level. 
this currently only worked when it's random level. 

## too many more new things I would like to see
- there was no collision detection in DMIMO so far. the controller was constantly active. that should affect the performance. 

- ~the user-hand model was not included in either simulation models. it could improve OL's stability. ~ (added 20191109) 

- ~now that "steady point" was introduced, maybe it's also good to see where it was. ~ (added 20191109) 

# 20191107
## achieved batch-wise constant disturbance level simulation
my head was clearer than yesterday. 
random level was achieved in *runLargeScale.m* b/c it set a random level for each QP within a whole batch. 
for a constant level, that's not necessary. 
every batch was done by running *runLargeScale.m* once, in which *disturb1DInit.m* was also initialized once. 
so the constant level was defined in *disturb1DInit.m* 
and just a note that, with random level, `setVariable` was used to setup `gridIn`. 
but with constant level, this was probably achieved by passing `pulseAmp` across variable workspaces. 

to save the figure from different level, I added *postConstLvSweep.m* based on *sumScript.m*. 
but I realized that it's not that useful and actually *sumScript.m* could get the job done with an additional plotting for-loop. 

currently the figures didn't really say anything much. 
the figures were quite weird when below 5 N. 
especially with 1 and 2 N, there were inf as well as stiffness lower than 10000. 
currently I considered stiffness below 15000 low. 
again, I couldn't really explain that. 
as the disturbance level was decreasing, more and more points increased to inf but some points decreased. 
the "increase to inf" part fitted my theory of high points but the other part didn't. 

judging from all figures, the ramp effect seemed always there. 
in a [previous log](https://github.com/easyt0re/piecewiseLinearModel#tried-a-few-things-and-thought-about-them) 
I used to find stiffness under 5 N rather flat, scattered. 
this was not visible in these simulations. 
I wondered what could be the reason behind failing to recreate those results. 
maybe it's the noise. 

## added another set of metrics w.r.t steady point
due to initialization and quantization, the TCP position before the disturbance was probably not the start reference point. 
it could be in/out of wall but it's almost never just on the wall (`z = 0`).
so there's actually this "steady point" that could be the "zero point" to check penetration. 
previously, the "zero point" was the start reference point. 
this was done in *postProcFcnLargeScale.m* and *sumScript.m*. 

the figures were definitely different but it's hard to tell which was better. 
and I believed both of them had problems in the calculation. 

added a helper function to plot each batch. *plotEachBatchInVizSum.m*

# 20191106
## preliminary tests on OL
with the unstable vibration aside, it was working. 
although the highest rendered stiffness seemed to be around 3000. 
this was 10 times smaller than DMIMO and this was the max stiffness. 
it's max b/c of the saturation I guessed. 
given a desirable stiffness higher than this, this would be what you really get. 
with a low stiffness of 500, minStiff was 240 something. 
so I thought it might be useful to see how "good" OL could render, meaning, maxStiff. 
this was added to *postProcFcnLargeScale.m* .

## a few changes in *postProcFcnLargeScale.m*
apart from the above, the way to find start/stop time index was updated to change with the delay and pulse width of the disturbance. 
this was b/c OL simulations had no delay while DMIMO had. 

another thing added was to visualize disturbance level. 
like I said in previous logs, I had a feeling that the disturbance level distribution affects the "stiffness" of this QP. 
so this was added just to have some future reference. 
this didn't take into account the noise. 

and of course, the *sumScript.m* was also changed accordingly to get a summary of multiple batches of simulations.

## constant level sweep
I failed. 
the current *runLargeScale.m* was supposed to run simulations automatically, with the disturbance level varying from 1 to 15 N. 
but it didn't. 
I could get the correct result if I ran manually. 
maybe that's a good way to do it but I really wanted it to run automatically. 
I felt like it's still the amplitude of the disturbance. 
b/c it's running in loop, the value could accumulate with the use of code like `a = a * i;`. 
either manually or fix this once and for all, I wouldn't have time today. 

when I read my code, I noticed that if `isRandomLevel = 0`, there was no code to set disturbance level in the `gridIn`. 
but then, how would my previous runs work? 
is it really that this was never triggered in the past? a hidden bug? 
I also enabled "TransferBaseWorkspaceVariables". 
that could be the real reason, variables bleeding across workspace.
alright, I would commit the code tomorrow. 
hopefully, everything would be fixed by then.  

## misc
- it's been a while since I last checked joint torque. I had never checked joint torque for OL. it seemed that with 0 gravity, prismatic guide, and rendering a positive z force, joint torque 6 was perfectly 0. 

- the "motor model" was mostly missing in my modeling. it's considered ideal. whatever torque the controller commanded, it would reach unless saturated. so the only motor model I had now was the saturation. 

- saturation was added to OL controller. 

- it might also be interesting to look at min of minStiff. after all, we were interested in min of the min. 

# 20191105
## came back to work on baseline 0, OL
I didn't really note down the current way to do simulation with OL. 
I still used *exeScript.m* but the name of the simulation model needed to be modified b/c they were currently 2 different files. 
to initialize the simulation, scripts needed to be run in this order: *userHandInit.m* and *studyBaseline0.m* in no specific order, and then *exeScript.m*. 
there was some strange blocks about saving `pose` into the workspace w.r.t time. 
instead of updating one set of values at each time step, the implementation seemed to save a "time series" into the workspace. 
if not reinitialized, the dimension of `pose` would mess up the next run. 
this was not a problem with the experiment but could be a problem with simulations running back to back. 
I actually had no idea why `pose` was saved to workspace. 
I always thought the new `pose` would be used in the next step and that's why. 
but after checking with the signal from the input `pose`, it's apparently not the case. 
for some reason, those new poses could only be obtained after the simulation was finished it seemed. 
so I suspected more that this "save to workspace" was only used in debugging. 
maybe I could replace it with a terminator block. 
but the current implementation was to add a `stopFcn` in the model to reinitialize `pose` once the simulation was finished. 
it's probably not even necessary b/c with *runLargeScale.m* I had some `clear` commands once everything was finished. 

a warning was also fixed. 
I used a "data store memory" block to pass inverse transpose of the jacobian matrix and previously it caused a warning. 
b/c everything worked, I didn't really look at it. 
it seemed that it was not working properly. 
there was not "error" in the simulation b/c I probably didn't really utilize this matrix. 
I forgot the purpose of it in the first place. 
the warning I believed was related to the read/write order of the value. 
I used to put the extra code in the for-loop but now I moved it outside. 
and now that warning went away. 

the model of OL had gone through changes as well. 
force feedback now would be turned off if there was a delay in the beginning of the simulation. 
but the delay was not kept. 
first of all, I didn't see the 0.4 mm penetration mentioned before, the reason to turn off force feedback. 
and to help get rid of the very small force generated by the very small penetration, a "deadband" was added in the simplified VE. 
TCP had to enter deep enough to trigger a collision detection. 
maybe by cleverly tuning this, it could also help to have a stabler rendering of the wall. 
secondly, even if there was a penetration after initialization, a delay would not really help to eliminate that 
b/c there was no controller for this initialization step, unlike our proposed position controller. 
thirdly, after the initialization, with a delay, TCP would actually move away from the wall, 
creating a distance for a free-space motion when the disturbance was applied. 
this was not what we needed right now. 
this moving away from the wall was probably created by the gravity of chain 3.
with gravity active, things started to fall and collapse. 
and with the smooth guide of the prismatic joint, it created this motion. 

with the last point, I finally decided to turn off gravity, a flag in *studyBaseline0.m*. 
otherwise, there would also be this force that pushes TCP away from the wall. 
now the XY stage was more about align the prismatic joint and less about gravity compensation. 
maybe it's ok to try passive XY stage now that there was no gravity. 
but I felt like there would be large deviation in x and y for TCP. 

b/c everything was smooth (0 friction), things could fly away into infinity. 
a limit on the prismatic joint was added to *userHandInit.m*. 
this limit was based on the fact that after `z > 30` for TCP, there would be an error and I currently considered it as hitting singularity. 
this created some motions that were not supposed to happen but could be cut off very easy. 

with all of these implemented, I could render some springs. 
it's not stable. the vibration would not go away but actually grow. 
but still I got 10000 N/m spring running. 
and the flying away at the end was b/c the disturbance went away. 

## a little bit behind schedule here
I spent too much time analyzing the data. 
I didn't expect it to be this hard to draw some good conclusions. 
and I probably spent too much time writing this. 
but this helped me think. 
but some of these could be comments in the code. 

## didn't really have the time to...
- check the constant level sweep from 1 - 15 N. I actually checked 1 and 2 as test runs. the disturbance was too small to create penetration, so some of the post-processing stuff could not be applied to cases like this. 

- think about my theory behind stiffness, penetration, and disturbance. after I check the sweep, I think I would have a better guess. 

- check the 4th and 5th 20 batches. I did them just to see more. 

# 20191104
it's important to understand your data. 
## ran 20 more of the previous 20 batches
as discussed in 1102, I ran these. 
I could have made many mistakes (I did and fixed a few) but it seemed that the performance was quite robust from the mean figure. 
after writing this, I also quickly checked standard deviation and it also seemed the same. 
(speaking of std figures. 
I was expecting some patterns but maybe I shouldn't.) 
however, I was still skeptical about this. 
maybe I should also check the distribution of force disturbance at each QP. 

this check and the high point check from 1102 were both b/c I didn't really understand the sudden changes in mean stiffness. 
what's so different there to have a high point (and it's really quite high)? 

## and then this happened...
I really doubted that the first 20 batches and the second 20 should look similar/the same. 
so I did a third 20 batches. 
and the high point changed. 
not very much but still a different point. 
this at least showed that what I got from the first and the second was not certain albeit probably with a high probability. 
this also confirmed some of my theories behind the high point. 
they seemed to be, indeed, existing and reasonable. 
then my hypothesis was actually every point on the upper half could be a high point, given the right amount of disturbance (and probably noise). 
this "every" was probably a stretch but at least there should be multiple high points, randomly or symmetrically. 

## ran 20 batches with constant disturbance level 10 and with noise
I should have done a test run first. 
it appeared that 10 was already enough to see the ramp effect. 
maybe I was being pessimistic but this was also bad news. 
the ramp effect was actually something to avoid in the beginning of design phase I presumed and something should not appear in the performance. 
(I had this discussion with Lei, he also agreed.) 
showing at 10 left a even narrower range for the disturbance. 
maybe I should run 1 batch for each level, w/ or w/o noise undecided. 

## a short discussion with Lei about these findings
they were quite small/trivial compared with the main goal here and they were a little bit too focused on the details (in implementation), I agree. 
the above was Lei's comments and he's probably right. 
but maybe I didn't make myself clear. 
I also forgot about my point in the discussion. 
I brought these to him not b/c I thought I should write about them but b/c these things questioned my methodology in my opinion. 
this went back to, again, the choice of how many batches, disturbance range, and the way to analyze performance (choosing stiffness and using mean and std). 

I would like to note down a few things w.r.t choosing stiffness. 
stiffness would be good if it's irrelevant to the disturbance level. 
but here it might not be the case. 
due to saturation, the rendered stiffness couldn't be irrelevant to the disturbance level. 
and how stiffness changed w.r.t an increasing disturbance level was also quite confusing to me. 
I lost myself again in thought in writing this but my goal of writing this was to say that maybe (of course) it's a good idea to mention about disturbance level range when talking about stiffness. 
this should be quite straight forward. 
in rendering a wall, if you pushing force exceeded max rendering force, then the mechanism wouldn't stand it. 
there's no stiffness what so ever. 

Lei also pointed out that though we didn't have a methodology behind choosing number of batches (previously 20 but now debatable), 
we could keep increasing until things converged. 
he pointed me to a paper he was writing and I thought I got the point. 
the value should converge and at some point it should be good enough, like a stop condition for classifier. 
however, my doubt was whether my "value" would actually converge and if so, could I just do some expectation calculation instead? 
my guess was this would be a weighted sum and the weight would be the determinant of the result. 
the other thing (idk the name) should be constant (approximately). 
the weight then should be the distribution of the disturbance levels across all simulations. 
if these were reasonable, there's no need to increase and wait for a convergence. 

## misc
- I had more thoughts on high point(s) but it's probably like what Lei said. it's not that important. it's probably a rabbit hole. 

# 20191102
## took a look at the high points finally
it seemed that the calculation was fine as expected. 
the high points in mean stiffness figure were there b/c they had a few very large values in the 20 batches. 
this could also be confirmed by looking at std figure. 
so I looked at each simulation result when there's a high (seemingly wrong) value. 
I plotted the penetration plot for the run and confirmed that max penetration got the correct value it's supposed to take. 
I also checked the disturbance signal (level + noise). 
judging from what I saw, I felt like these high points were easier to reach when the disturbance level happened to be around 5 N (low) and the QP on the upper part of the wall. 
I guessed this went back to the discussion of what is the proper range of the disturbance level. 

~I could do another 5 - 15 random with noise for 20 batches and analyze this alone or together with the previous 20. (done 1104)~
I could also do no random level with integer 5 - 15 with noise, each for 20 batches. 
this I could start with only 5, 10, and 15. 
10 should be the first to run b/c it seemed promising. 
if my guesses were correct, 5 would give high points while 15 ramp effect. 

# 20191101
## tried a few things and thought about them
tried 15 mm U/D/L/R away from origin as OP and with 5 N disturbance. 
they didn't show anything very exciting. 
the stiffness distribution was barely changing. 
at this level, the ramp effect was barely showing. 
there seemed to be more circle and oval shapes in the plots. 
one thing Kjell mentioned was that the linearization effect might not be as simple as a circle around an OP. 
this was unfortunately true and added some complication to the identification process. 
the current way was done by me eyeballing. 
there might be a way to filter the image a little bit, to smooth out the noise but I didn't want to process the data too much. 
moving the OP up and down did decrease/increase the performance of the system given the fact that the upper part of the workspace had a better performance on its own. 
moving the OP left to right didn't really change anything. 
subjectively, I felt like, with the left one, the "pattern" was moved a little bit to the left. 
but the shape was really not that obvious given the reason mentioned above. 

I also did the 15 N only with the up/down. 
again, nothing much. 
but this brought up the next point: was the random level ok? should they be compared together? was 20 batches enough for this random level? 
take the 5 N and 15 N as an example. they actually showed different varying patterns over the workspace. 
15 N had the ramp effect and 5 N was rather flat I believed. 
I imagined this would be bad for the `std()` later. 
at least there should also be some kind of ramp in std plots. 
but it didn't happen. 
that's why I thought 20 batches might not be enough to demonstrate this. 
I really should try to find a methodology to find a proper size rather than choose 20 arbitrarily. 
apart from stiffness, there was also the max force TAU could generate. 
maybe 15 N was already out of range and shouldn't be discuss or included. 
maybe we should focus more on a range where TAU wouldn't saturate so often. 

in general, I guessed I had to do these plots with a fixed scale, otherwise they were even harder to compare. 
`[1.5, 3]` seemed to be ok. 
and maybe I should get back to running smaller batches. 

# 20191030
## meeting notes
Lei confirmed my thinking of what I was expecting to see: 
linearization making the area around the OP have a better performance - higher stiffness. 

suggestions for the draft: 
- Kjell has a separate document for this

- turn off gravity or somehow separate the load by gravity and that by rendering to see the effect of linearization

- add more theoretical stuff for the discrete time design part like discrete time state-space model, how to do `c2d()` in theory and some book referencing. 

- add more figures and equations for the simulation setup part. now the reader might not follow the whole thing. 

## run simulation, run
I felt like I didn't care about running anymore. 
it's somehow cheaper than before. 
thanks, Xinhai.

the batch used to be 11x11. 
I changed into 21x21 and, without a pause, 51x51 to have a higher resolution. 
whether or not this was necessary was still debatable b/c it seemed to show something more but also with a lot more noise in the figure. 
the main trend was still the same in 11x11. 

I tried with different choice of (one) OP but for some reason I changed the constant level of disturbance to be 5 N. 
this might be too small for TAU in a sense that it's away from saturation limits - the ramp effect (stiffness increases with y coordinate) was not showing. 
I guess this proved the ramp effect was the result of saturation in a weird angle. 
but I guess this was also not enough to see what I wanted to see. 

there were a few things to try. 
a constant disturbance level of 15 N should be checked. 
with random level disturbance, maybe more tests were needed. 
or there was a chance that the range of the random level was not good. 

## misc
- realized I could do some simulations with only 1 OP but at different places instead of the origin. created a bunch of *saveControlTest?15.mat* files for different OP on x and y axis. 

- (I could) turn OL force feedback off at first to that 0.04 mm penetration mentioned in yesterday's log. 

# 20191029
## implemented the OL baseline
a version of the open-loop (OL) force control type haptic rendering was implemented. 
it seemed to work. 
it's in a separate file named *disturbTestBaseOL.slx*. 
with the findings and thoughts after building the user hand model, an XY stage was added to the original *disturbTest.slx* to become this new file. 
later, I realized the important parts are the active XY stage and the passive prismatic joint that's used as a "guide". 
this "add-on" could actually work with previous position controllers though the joint torques were weird looking b/c the motion of the TCP was constraint to translation along z axis only. 

this was not the purpose of course. 
it also worked with the OL controller. 
the controller was mostly from *cosim_unified_system_asm.slx* from the "tau without dspace" folder, another project that for some reason was private currently. 
small changes had been made and currently it seemed working. 
this had some old code in it. 
the old code, and probably most of my code, as I mostly only borrowed from old code, was written with mm as the unit. 
this should be kept in mind. 

current status/problems with the implementation
- due to quantization (I think), TAU would initialized 0.04 mm into the wall. it's better to fix this. 

- that's why there's no delay before the disturbance anymore. otherwise the small push from the wall would mess up the whole system when there's zero friction. 

- only the case at the origin was tested. it was not stable. the TCP would "vibrate" (bounce off the wall) more and more until it crossed singularity. 

- a full user-hand model might enlarge the stable range of rendered stiffness. the current value is 1000 N/m, not too much

- there's this "save to workspace" with `pose` that seemed unnecessary. needed to find a way to fix this. 

- b/c of the unit difference (m vs. mm) and the borrowing of code, things like `L1` could conflict with *studyBaseline0.m*. needed to find a better way to do this so that every controller could be initialized properly. 

- b/c TCP was bouncing, I didn't really check if it worked. I remembered I had a 10 N disturbance and the penetration looked about right. needed to look closer. 

## misc
- found the gear ratio in old code and now it's in *studyBaseline0.m*: for 1, 3, and 6 it's 10/100; for 2 and 4 it's 10/35; for 5 it's 10/130. this coincided with what I found before in a report or some other written paper form. 

- also copied over functions needed: *FTtransform.m*, *skewVec3.m*, and *SM_FK.m*. the first two were for dependency reason. the last one was just for figuring things out. 

# 20191024
yesterday was 1023, which was 1111111111. 
## built a simple model for user hand
the user hand cannot be modeled as either pure motion or pure force. 
the logical thing was to model it as a mass-spring-damper system. 
of course it could be more complex but let's start from here. 
this was why I didn't really do it in the first place. 
but it seemed rather easy now that I did it. 

I started *testUserPush.slx* for simulink model building. 
later, it should be able to work with a version of *disturbTest.slx*. 
maybe I could even replace the current disturbance signal with this "human hand model". 
I also started *userHandInit.m* to initialize all the params needed for this part. 

currently, the test was fine when the wall is with a stiffness of 1000 (N/m). 
it also showed some small vibration (well within +/- 1 mm) when the stiffness was 100000. 
this was promising b/c it showed that it was still OK when the stiffness of the wall was not so high. 
but when it's closer to an ideal stiff wall, vibration started. 

the params for the hand model were from a paper I searched currently. 
at least I had something to reference to. 
the stiffness as well as the damping coefficient were both quite important for simulation w/o vibration. 
the model of the wall was currently just a spring. 
adding other stuff like damping would also improve stability but it's not the current goal and a wall shouldn't have damping. 

everything was built based on the reference point, the origin of the task space. 
there was a passive XY cartesian stage to have good alignment of all z axes (the world, TCP, hand). 
~whether or not this should be passive was still undecided. 
passive wouldn't work. it would move away. 
active wouldn't work. it would interfere with the motion of the TCP. 
the ideal way to go was: it was passive when started. 
TCP probably initialized at the origin and moved to a query point, maybe even a point that's not on the wall but a bit away from the wall. 
then it became active so that the x and y coordinate of the TCP is "fixed".~ 
ok, that's probably wrong. 
this cartesian stage was there to ensure the x and y coordinate, in other words, compensate gravity when the controller couldn't. 
~that meant this should only be used for the baseline case. ~
so what might really end up happening was that I had 2 models for this simulation. 
one was the one I had, for the controller we proposed. 
in this one, the stage should be passive so that there would be no interference, while still having good alignment. 
the other one was for the baseline (currently it's the open-loop force control) ~and this XY cartesian stage was used~. 
the stage should be active and how to initialize would be a later problem to discuss. 
correcting myself all the time. 
too many new things and new ways to implement. 
I was in a hurry so I didn't really think this through, you would see. 

whether or not this would work with what I previously did was still under the question. 
but I thought it should. 
~this was not really needed to build a baseline but for some reason I started to do this.~ 
the reason why I started this instead of a real implementation of the baseline could be found in the previous paragraph. 
it's needed, otherwise the baseline wouldn't work. 
ok, the XY stage was needed but the hand modeling was probably not. 

## made a script for generating the stiffness figure
for some reason, I forgot to mention I made a separate script for this figure. 
it used to be a bunch of commands in the MATLAB history but as I moved forward in writing, this should be a script. 
maybe I didn't mention this only b/c it's not finished yet. 

# 20191022
wasted 1 day on baseline 
## implementation on baseline
I read more about this MMA paper. 
currently I believed I got most of the implementation right. 
it came down to 1 MATLAB command `sdhinfsyn()`. 
this was the one they said they used in the paper. 
unfortunately and strangely, the MATLAB documentation for this was quite poor. 
along the way, I found commands like `makewight()` and `augw()`. 
they seemed to have changed quite a lot and the documentation was outdated somehow. 
again, very strange. there's even error I believed. 
I didn't know what I did wrong and I was out of things to try. 
b/c there's not too much time, the next step would be to give up and try open-loop force control. 

a few things about the paper. 
I had always thought that it's not a very good paper. 
maybe there were some misunderstandings but it's not straight-forward, it's all over the place, and some of the things they did for the paper I didn't agree. 
it's actually like a typical paper I would write: no highlights, chaos, stuffing stuff just to make the pages. 
but they were really good at write more words. 
the current problem for my draft was I had nothing to write about though I kinda did many things. 

# 20191021
## new things to do to make the draft richer
things showed up in writing. 
- [ ] do 1 batch of simulations with the same level of disturbance to see if joint 6 torque needs to be larger when y coordinate is smaller
- [ ] find a really good way to rotate the axes so that the xoy plane is in a "wall" position, not a "ground" position (it's more of an orientation thing actually)
- [ ] do a finer grid b/c I suspect the current resolution is not enough to show what we care
- [ ] take a look at the high points mentioned last time to make sure the numbers are correct

# 20191018
cursed by the knowledge of the future. 
## made related changes to DMOPMIMO
edited *genMultiOPs.m* to generate gains for DMOPMIMO. 
generated and saved all the gains as *saveDMOP.mat*. 
added related parts in *exeScript.m* as well as *disturbTest.slx*. 
tested a few points by running *exeScript.m*. 
(cleaned signal logging for later *runLargeScale.m* to save stuff)
the points on the edges were a little better b/c it had more OPs. 
the points on the boundaries of the switching were terrible. 
(when I first realized we would do MOP later and the supposed benefit of it, I was also wondering the drawback of this other than saving a few look-up tables. 
then I realized the weak points shifted from the edge points to the boundary points. 
and now I knew how bad when "nearest" was used for switching) 
`y = -12 or -13` were already having peaks larger than DMIMO under disturbance. 
`y = -12.5` had oscillations. 
not to mention `(x,y) = (12.5, -12.5)`. 
this was something that must be fixed, otherwise there's no point of doing MOP. 
this might not show if I ran *runLargeScale.m*. 
but I definitely need to at least address this in the future. 

as I said in yesterday's log, I just wanted to peek a little bit about what's coming. 
this was not supposed to be what's worrying me currently. 
but now I felt like I couldn't care about anything else. 
alright, alright. 
I had to let go of this and let the future me worry about this. 
I stopped at running *exeScript.m* with DMOP. 
*runLargeScale.m* was probably not ready for DMOP. 

## looked at the newly generated simulation results
the new result was named *saveSumDelayv3.mat*. 
this was with 0.5 s delay, the delay had no noise, and the way to get instant stiffness was also a bit different. 
the mean of the wall plot (another name for the distribution plot) looked the same as v2. 
with y increasing, the stiffness increased. 
the std plot looked smooth and it probably should. 
there's no obvious tendency. 
there were a few (1 or 2, maybe up to 4 out of 121) high points. 
and with the high points gone, the ratio of std over mean could be less than 10%. 
there's also no tendency in the ratio, which was a bit strange. 
compared to the "common" points, the high points were a bit odd.
maybe it's better to check them out before moving forward. 

part of the reason to turn off noise while the delay was to get better results for `minInsStiff`. 
but it appeared that the period after the disturbance went away also affected the number. 
took a step back, what we really wanted to capture was the penetration when the disturbance happened. 
so I limited the time period for the search to 0.5 - 1.0 s. 
maybe I should also do this for `minStiff`. (I didn't) 
results were generated w/o doing the same to `minStiff` and named *saveSumDelayv3.1.mat*. 
with this modification, `minInsStiff` looked a lot like `minStiff`. 
it also had the high points in std plot. 
`minInsStiff` was always lower than `minStiff` in mean plot, as expected. 

## misc
- b/c we decided to shift away from the SISO method, SISO related code would be slowly commented out and removed. 

- there might be some naming issues between OPs and QPs. check this before ran *runLargeScale.m* on DMOP. 

# 20191017
had a meeting and talked about responsibilities. ok. 
## more on generating stiffness distribution plot
I finally had a name for this plot. 
with the problem of (9,6) I realized I should apply the disturbance when the system was stable instead of apply when it's still trying to stabilize after initialization. 
I added a 0.5 s delay in the beginning and ran 20 tests. 
*disturb1DInit.m* was modified for this. 
I got the plot. 
it looked nicer but then I realized I should only check 0.5 - 1.5 time period for penetration. 
so *postProcFcnLargeScale.m* was modified to "cut out" the 0.5 delay in the front. 

after this modification, the mean stiffness distribution seemed right and very "smooth".
with the increase of y coordinate, the stiffness seemed to go up linearly. 
this was something quite interesting. 
up until this point, I wasn't really expecting anything anymore. 
however, this seemed to show that there was just some performance distribution issue in the workspace in general for TAU. 
I don't think this is an effect of the linearization. 
it could be the fact that when y coordinate is larger, TAU could exert/withstand larger force on TCP in z direction. 
so I guess what I was hoping was to have a distribution that has a better performance around the origin than far away. 
if that's the case, the idea of multiple points linearization would make more sense and could be the next step. 
now, I could still argue that what happened with (9,6) was the motivation for MOP. 

maybe I should try to run DMOPMIMO first and see what would be the improvement. 
then I could have a better discussion and a strong base for the next paper. 

## misc
- from now on, name every folder for results starting with "save". modified *.gitignore* to ignore all folders with this beginning. 

- also realized after adding the delay, there was noise in the delay/"should have nothing, waiting to be stable" phase. modified *disturbTest.slx* and added a step as an on/off switch. tested this in the *testNoise.slx* first. 

- notice: for the current code, with the `isRandomLevel = 1`, if I only run *disturb1DInit.m* and not *runLargeScale.m*, the disturbance level is always -1 N. this is the case when I run *exeScript.m*. 

# 20191016
wow, traveled into a little bit into the future back there. 

## reflection on the discussion with Lei
the discussion ended up mostly around the states of the system. 

we discussed moving states from the current joint space to task space. 
the intuition behind this was to have task space values as state vector when using LQR so that we could constraint z while let x and y free. 
it seemed that there were some issues doing that. 
b/c we were doing full state feedback, the states had to be "observable" (should be the correct word). 
I wouldn't say the task space values were not observable but it's hard to observe them. 
another thing about changing the states was that it might be hard to have meaningful values or representations. 
if we chose TCP pose (xyz and roll pitch yaw), what's the meaning of the dot of these values? 
if we chose cartesian and angular velocity, what's the meaning of the integral of these values? 
these problems focused mostly on the angular part of the values. 
I'm sure there must be ways to do this but I don't know how. 
mixing task space position with joint space velocity was also not valid b/c you have to have the values and the dot values at the same time. 
the conclusion of this discussion was maybe we could do something better. 
however, for the current knowledge I have, doing control in joint space is the best I can get. 
of course, we could still do things in joint space and map it to task space. 
but the mapping was not straight forward and made this LQR thing less intuitive. 
the whole point of moving to task space was to make things intuitive. 
this reminded me more and more about virtual fixture/active constraint. 
they have to have something to project the object in task space into joint space in order for joint space control methods to work. 

we also discussed about the definition/formulation of the problem. 
more specifically, the states and IOs of the control plant. 
most things appeared to be pretty settled. 
the input was the motor torque. 
the output for now was the encoder reading. 
anything related to the user was the disturbance but I didn't really specify where it entered the system in my block diagram if I had one. 
the problem or the crucial part was the state(s) and how to choose them to do implementation. 
the output was something measurable/observable. 
I don't really think the two should be interchangeable but probably it is here. 
the states for implementation must be something observable given the output. 
for example, given encoder reading (joint position), we could get joint velocity. 
we could also "estimate" TCP pose based on encoder reading but forward kinematics was a bit tricky here. 
and even if we could, the problem discussed above still remained. 

a side point we discussed was about making everything about error, the point suggested by Yuchao. 
Lei's current idea was that the current "structure" was enough at least for now. 
this change would actually change the control logic of the controller among many other important things. 
it's not as non-trivial as what we did with changing the auto-generated states to desired ones. 
so this point was dropped indefinitely. 

# 20191015
## discussion about the initial wiggle
after discussion with Lei and Tong, what happened at (9,6) was basically confirmed to be due to linearization. 
and the wiggle seemed to be inevitable given the fact that we are doing linearization. 
the solution to this could be multiple points, which should be our next step. 
there could also be other ways but it seemed that it's supposed to happen when using basic linearization methods. 

another way I came up with was to change a little bit about the simulation. 
we could add a small motion in the front of the whole simulation. 
the simulation always initializes at the origin, making the least wiggle. 
use `stopp_pose` to define the position where the disturbance actually happens. 
before anything happens, TAU would initialize at origin, move to another point, and stop there, stable. 
then, at that point, invoke the disturbance. 

as I was typing this, I realized what I did here was just to invoke the disturbance when the TCP was stable, instead of when the "time" started. 
so all I needed to change was probably a 1 s wait/pause for the TCP to be full initialized/stable and then apply the disturbance. 
so the *disturb1DInit.m* was modified with a 0.5 s delay in the beginning. 
the whole simulation grew to 1.5 s, with 0.5 s nothing, 0.5 s pulse, and 0.5 s nothing. 
a pre-test showed that now the effect of the pulse coming and going seemed to be more or less the same. 
I would run 20 tests with this later and re-evaluate.

# 20191012
## checked 0 disturbance cases to look into the (9,6) case
it seemed that everything was due to linearization and compensation. 
with 0 disturbance, initializing at the origin showed almost no vibration and the only wiggle should be coming from the encoder resolution. 
then, negative y value (initializing at a position below x axis, y = 0) yielded a positive peak in z direction at the very start of the initialization. 
and a positive y gave a negative peak, respectively. 
b/c there's linearization and at the very beginning of the simulation there was no "previous reference", the controller was probably thinking that TCP was at origin but it's not. 
the controller worked solely based on the compensation value from the linearized point. 
with x axis (y = 0) as the dividing line, the peak was probably caused by the "urge" to go to origin but later realized, "OK, I'm actually not there and I shouldn't go there." 

the disturbance on the other hand, a push against the wall, had the tendency to cause a negative z motion. 
these 2 motions/tendencies would both contribute to the final deviation/penetration the user would feel. 
when below x axis, if the positive peak was even larger than the negative z motion caused by the disturbance, it could result in a non-negative deviation. 
and that's probably what happened at (9,6). 
with the largest disturbance (15 N), the ultimate penetration was still quite small, not to mention the lower bound was 5 N. 
and if we took a look at the points above x axis, I imagined the already negative peak would make the penetration worse. 

since this was probably happening along y axis, I also check x axis. 
b/c if this was the result of the linearization, all points should be affected. 
looking at the result, maybe this direction was somehow not as bad and a little bit different. 
all points on x axis seemed to have negative-positive-zero wiggle along z direction, no matter left or right side of the y axis. 
however, it seemed that points on the left had larger (absolute value) negative peaks than the positive ones. 
the points on the right seemed to have positive peaks a little bit larger than the negative one, if not similar. 
all peaks considered, the range was within 0.4 mm, unlike the peaks along y axis (within 1 mm). 
this probably explained why y axis was not a dividing line for the calculated stiffness. 

so it seemed that there really were some influences from the linearization. 
and the fact that this "symmetry" was not really w.r.t the origin but rather the x and y axis was probably b/c of the structure of TAU or even the choice of motors. 
but these still felt like imagination. 
it's not a perfect explanation. 
but this was my best guess for now. 

## examined post-processing the result of 20 simulations
currently, the simple fix for (9,6) was to `ans(9,6) = ans(8,6);` b/c (8,6) most of the time happened to be the second highest value. 
currently, (9,6) was more like an out-lier and this was the way to eliminate it to have a better look at other points. 
this hot fix was applied to any values that produced a "heatmap". 
`std./mean` was plotted and 45/121 was below 0.1. 
maybe this was already good enough. I cannot tell. 

the points along the negative y axis were still different from others. 
they were probably 10 times higher than $x>1$ and $x<-1$. 
again, I don't really have good explanation for this. 

## misc
I don't know if this is a feature or a bug for MATLAB. 
when I used Signal Data Inspector and when I clicked on the signals in a descending order, the signals started to change.
it's not all of them. and I guess if I continued, it would cycle through all the signals. 
I can see the potential of this feature/bug but the important thing here is how the program could tell what I really want to do.
in my case, I don't really want to cycle through signals. I just want to see them. so it's buggy. 

# 20191009
## the meeting notes
the contribution of the discrete time paper again narrowed to the idea of doing haptic rendering with position controller. 
I had no problem with that but Lei seemed to suggest once again to put real experiments in this paper. 
I also had no problem with that but I considered this a huge change in writing and research. 

the reasons why I avoided real experiment:

- I didn't think the hardware was ready.

- there was still much work to be done for my controller to work on the real hardware. 

- I didn't really think about the real experiment or how to really do it. I still believe I cannot copy everything from simulation to experiment, though both supervisors think otherwise. 

- the modeling error between the simulation and the real machine was too huge to handle by the controller. basically, I'm saying probably it's not going to work.

I'm lost again moving forward. 
this happened to me after half of all my meetings. 
because of this, I question the function of the meetings. 
at least I should set goals for myself next time when we have meetings. 

- [ ] a figure of how the grid of QPs is located in the paper showcasing the OP, the QPs, and the workspace

- [ ] SISO is not the focus anymore, try new baseline

- [ ] maybe plot `std/mean` to have an idea

- [ ] check the reason behind all positive deviation at (9,6)

## checked (9,6) for negative stiffness
previously, a closer look showed that the deviation was always positive, making the calculation of the stiffness wrong. 
I ran this simulation alone w/o any disturbance. 
it showed that in the beginning there was a flicker in z direction with a peak of positive 1 mm. 
what's more, the TCP actually stabled at a positive z value. 
it cannot really reach 0. 
so maybe it's better to check all points with no disturbance to see the actual position of the TCP.
this is due to quantization I think. 

with roughly the same disturbance (7 N) turned on, this peak was smaller but no negative deviation appeared. 
hence, no penetration. 
with even a 15 N disturbance, the upper limit, the system showed a very small negative deviation, meaning the stiffness could still be very high but in a correct way. 
at this point, I couldn't figure out why the "performance" (stiffness) of this point was so outstanding. 

## the idea of restarting all over again
Lei gave me the impression he was feeling "you could definitely do better than this. why don't you do it?" 
I'm happy and sad that he might feel that way. 
I'm happy to see potentials and I'm sad b/c I don't really feel like it works that way and there were things I've done wrong, making this not as potential as he thinks, maybe. 
so here is a list of things I could do differently, maybe right away. 
thus, this could also be a **TODO** list.
- [ ] do everything in task space instead of joint space

- [ ] have different constraints on different directions

- [ ] have well-planned-out I/Os for the control plant and differentiate states, measurements, outputs, and so on

- [ ] distinguish joint torque and motor torque (the modeling of gear ratio)

- [ ] ~do everything with error instead of position~

after listing everything here, it doesn't seem like much and maybe it IS something that I can really do. 
now, although some of this should be repeating old notes, I would like to list the reasons behind these re-dos.

the first point is to enable Lei's idea, which is actually the second point. 
apart from that, b/c we are using LQR, working with task space is more direct and intuitive.

the second point is the main point. 
this enables constraints on z direction to stop TCP on the wall and no constraints on other directions so that it can move freely, closer to what we are supposed to do.
ultimately, it's related to the surface normal of the wall we are rendering. 
thus, essentially, the Q matrix, the weights on correcting deviation, is related to the geometry of the wall, a property of the wall. 
so instead of a stiffness of that wall, we save the weights associated to that wall. 
and when in collision, these weights would be loaded to the controller to be activated.
but on top of this, there is still a "switch", which I guess it's the collision detection flag in this case. 
I don't know how to do a unilateral constraint with LQR (correct positive error but not negative one).

the third point has to do with better definition of things, to make sure that the control plant is close to the physical machine. 
apart from the formulation of things, I would like to specifically point out that there is no joint velocity sensors installed. 
the only measurement currently available is the encoder reading. 

the fourth point is rather concrete. 
for some reason (lazy) I didn't incorporate gear ratio in the current model. 
the thing I'm controlling is joint torque, not motor torque. 
this would affect the torque and the encoder reading. 

the fifth point was brought up in the discussion with Yuchao and Yu. 
though I don't fully understand, formulating it that way makes it closer to the form of tracking problem. 
then some of the tools from that area can be of use maybe. 
regardless, it seems to make the form more elegant and straight-forward for some analysis.

# 20191008
## further implemented Monte-Carlo in *runLargeScale.m*
the current script ran in a way that 1 simulation / 1 batch ran all query points once. 
based on this, there were 2 ways to randomize disturbance level.
for each batch, generate 1 random number; or for points even in the same batch, generate a random level.
the point of the first way was that for the same batch, the level was the same.
since we were supposed to do Monte-Carlo, keeping the level in each batch the same wasn't really necessary.
so I implemented the second way in a not so clean fashion.
however, this also made it "useless" to see 1 batch in 1 plot (maybe) b/c the levels changed.

while we were here, let's take a closer look of the Monte-Carlo process. 
the purpose was to show mean and variance to a certain property. 
in my opinion, this property should be at least be constant in theory. 
in other words, it really has a mean to begin with. 
the reason why I brought this up, was that I always suspected "this property" varies under different levels of disturbance. 
if the property in question is the max deviation/penetration, this is probably the case. 
in this sense, maybe this property/metrics is not ideal. we should choose something "normalized". 
if this property is stiffness, maybe it's better but I still suspect that it also varies with the level of disturbance. 
to look at this in another way, we are rendering a infinite stiff wall. 
max penetration shouldn't change w.r.t level of disturbance. 

## stressed more on stiffness in post-processing
given the discussion above and all the deviation vs. penetration confusion, stiffness related metrics were chosen to be reported later.
this time we carefully chose those positive stiffness. 
I was thinking about making the visualization part a helper function.

## preliminary findings from Monte-Carlo simulations
used Xinhai's desktop PC to speed up the process. 
only looked at stiffness related metrics. 
in general, the values varied a lot, judging from the variance.
the min was at $10^4$ and the max was at $10^5$. 

in details, the point (9,6) was a bit out of place, stiffness too high. 
was there something wrong that led to this high peak or the peak was supposed to be?
at first, I thought there was singularity, meaning the calculation was off. 
but now I felt like maybe nothing went wrong. 
it's in the middle of a few points and actually a few better points. 
the neighborhood seemed to show a rise in the stiffness. 
but this single point was probably too high. 
if nothing was wrong, maybe try log axis?

one way to find out was to run multiple random simulations at this point. 
and I could maybe swap out the "bad values" if they were indeed problematic. 

a closer look at (9,6) batch by batch showed negative stiffness. 
this was definitely "wrong". 
to single out this simulation, `idxQP = 28`. 
a bunch of useful commands were still in command history instead of in script. 
basically, for a simulation that went wrong, there was no penetration. 
deviation in z was always positive. 
this could be fixed in the post-processing script but I also needed to make sure this "all positive deviation" was supposed to happen. 

## misc
ready to move and saveFiles were currently not pushed to git by .gitignore

# 20190919
## misc
-  cleared most of the data logging in MIMO and DMIMO controllers. didn't change the other 2 at this point b/c they weren't used. this was the attempt to shrink the data saved after parallel simulation.

- deleted inport/outport in *disturbTest.slx* b/c they are not used.

- commented out all the to workspace blocks only in DMIMO controller b/c they are not used.

# 20190918
## important points of the meeting
- [x] do random disturbance level (amplitude) instead of a white noise. white noise could be kept but random amplitude is more important

- only do DMIMO for now

- allocate time for subtask (development, simulation) and plan

- [x] check stiffness and they should all be non-negative

- remember to add a plot when linearized at a different point

- [x] add another stiffness: instant stiffness, stiffness at each time point

- [x] save disturbance and noise

- [x] choose what to save carefully

# 20190916
## tested white noise block
this block was governed by noise power and sampling time. 
I couldn't and didn't try to find a relationship between the outcome and these two.
the point here is that actually these 2 affect the amplitude of the noise at the same time.
`[0.001, 0.001]` yielded a noise between 4.
`[0.001/4, 0.001]` yielded a noise between 2.

# 20190912
## added noise to demonstrate robustness and sensitivity
currently the noise and the "randomness" was originated from 2 things.
there could be sensor noise. 
this could be done by changing encoder resolution.
the other was when I applied the force disturbance at TCP.
these could be white noise and could be applied at the same time.

the "no noise" case was still kept.
for the "noise" case, maybe we could run 10 times per point.
in implementation, we could run the whole thing (*runLargeScale.m*) 10 times with noise.

# 20190910
## misc
- changed 5x5 to 5x3 to debug the code better.

- noticed some of the variables were not initialized every time, hence there could be some errors. could be fixed later.

- added `checkidx` to understand and check if everything matches.

- added all the `_Viz` variables to better visualize points on the wall.

- changed force from 20 to 15. 20 was too large and with a 3x3 grid, many points ended up in singularity. this was probably due to saturation. temporarily changed to 15.

# 20190909
## started "large scale" work
started the development of *runLargeScale.m* to run a large number of simulations in parallel and save everything for post processing.
the controllers (gains) should be saved previously and loaded for the simulation instead of calculating them every time.
the simulation is done with `parsim()` and `io` is defined now mostly for different query points.

a post-processing script was also developed. 
currently, it's for spotting singularity and making sure the simulations were correct.
a real analysis should be developed later.

## modularization of the system
frankly speaking, I don't have the correct idea of modularization of a control problem.
I should have clearly defined all the I/Os for each module and setup boundaries for them.
it's a bit late b/c so much had been done already. 
but at least I could try to keep that in mind from now on.

to compare different controllers, other modules, like the control plant should be the same.
in my case specifically, the encoder model (quantizer + zero-order hold) should be a part of the control plant model.
the angle compensation and other things should be done after this and outside of this.
also, reference signal should be the same.
Lei also pointed out that I shouldn't quantize reference signal and angle compensation in the controller b/c controller should be relatively independent of the resolution of the encoder.
this controller should work for a wide range of encoder resolutions.
the controller doesn't need to be changed with a different resolution.

thus, these changes were saved in the new version of *disturbTest.slx*. 

## misc
- updated all the controllers in *disturbTest.slx* to have encoders (quantizer + zero-order hold) and kept the name. the old version was saved as *disturbTest_bak.slx* for backup and quick roll-back.

- corresponding to the adoption of the encoder model, the way to plot joint angles in *plotScript.m* was also updated.

# 20190905
new beginning (IMPORTANT)
## yesterday meeting recap
3 main things needed moving forward with this discrete time extension (DTE) paper
### thorough performance evaluation for the whole workspace
this point means testing (query) with a lot of points on the wall, with noises and other things included. 
this is to show robustness and sensitivity.
this shows an overall image of the controller performance instead of just one point.
this also includes my discussion of the performance drop away from OP.
this needs to be done with a larger scale, running 100+ simulation for 1 case.

**TODO:**
- [x] a `parsim()` (maybe) script to run things in parallel to speed up
- [x] choose a set of varying parameters and how they vary
- [x] maybe introduce some randomness into the simulation
- [x] save everything (parameters and results) in case of later analysis
- [x] automate the evaluation process with metrics
- [x] metrics list: RMS, rssq, stiffness, and maybe something in time domain, in joint torque

### apply this to ADAMS in the absence of real experiment
this point is not really needed.
but b/c I cannot do real experiment right now, this serves as what happens when there is a modeling error. 
this can be updated later b/c right now I don't have a concrete todo list. 
this can be done either in co-simulation or directly in ADAMS depending on the capability

### find a baseline to compare with
this point is hard to do b/c right now I don't know how to implement any of the current methods from other group myself.
there are some potential candidates though.

### other things like: 
- perturbation in other directions

## misc
- added *saveAnimation.m* to save animation with script

- modified the order of plotting to push noisy signals in the back

- changed "MIMOMOP" to "MOPMIMO" so that all controllers have similar/the same ending. this was better for saving variables using regular expression.

- added *matlabCodeSnip.md* to accumulate useful/not yet in-use commands and scripts. a list of things I wanted to solve was also included.

- added save mat file in *saveMultiFigs.m*

- updated Q matrix gain in *linScriptJS.m*

# 20190901
## copied 2 more controllers to *disturbTest.slx* 
copied from *DcontrolDemoLMwIAW1D.slx* and *controlDemoIJ1D.slx* to make DMIMO and SISO controllers. now, this is really a "one file rules all" situation.
the *exeScript.m* and the *plotScript.m* were all updated accordingly.
now the plots had designated color and the joint angle for DMIMO was a bit different from others just to show the "discrete nature" of things.

## parameter search again
on [20190612](https://github.com/easyt0re/piecewiseLinearModel#tried-to-sweep-for-good-q-matrix-again) I tried to search for new "gains" for the controllers b/c I was trying to do discrete time and encoder quantization.
the goal was to "relax" the control a little bit so that there was enough "error" for the encoder to pick up.
on [20190701](https://github.com/easyt0re/piecewiseLinearModel#again-note-the-setup-for-the-simulations) I made notes again about the "gains" used in both MIMO and SISO methods.
I remembered that I showed this result to my supervisors and it was quite promising.
for the last a few days, I had been trying to recreate the result but it didn't work.
MIMO and DMIMO worked fine but SISO was almost definitely better.
this was not what I remembered and I couldn't explain.
I did another sweep in for the Q matrix but it didn't help much.
maybe instead of making MIMO better, I should relax SISO a bit. 
but I got no reason to do so and I didn't really know how.

however, DSISO was probably not going to work.
it seemed that for MIMO, "c2d" was OK and the performance was comparable.
however, for SISO, "c2d" (both discretization and quantization) pushed the system to instability.
and it didn't end here. read on.

## test performance "far" away from OP
SISO was only "linearized" at origin.
MIMO could be linearized wherever but in current case also at origin.
with a `y = +25 mm` offset, it seemed that SISO outperformed MIMO, less error and less saturation.
originally I was going to show the performance drop as the "query point" moved further away.
this still holds within MIMO but I couldn't explain how SISO wasn't affected that much.

## misc
- added TCP pose plot, currently more interested only in z position plot

- made the simulation longer b/c from some plots it seemed that the system needed more time to settle

- updated *saveMultiFigs.m* to create folder when it's not there

# 20190824
lost again in my own files, both as in "I don't know which file is for what" and in "I don't know what I should do next"
## revisit discrete time implementation with IJC
at least from my previous logs, I didn't see anything about doing IJC with discrete time. 
*DcontrolDemoIJ1D.slx* was saved based on *controlDemoIJ1D.slx*. 
added zero-order hold and quantizer from the MIMO counterpart. 
to my surprise, there was never really a file for discrete IJC for disturbance. 
and the result was that it didn't work. 
last time I did comparison between time domains, I probably only did MIMO. 
is "SISO is not working" good enough? I don't know. I also don't know why it's not working. 

## recap about meeting 0821
the deadlines for 4 journals (LOL) were set at: Sept 22, Nov 15, Mar 1, and May 1. good luck.

maybe I'm just stupid but Lei finally gave me the correct way to do the controller. it's a position controller. so far, the Q and R matrix made sure that the difference (error) between desired position (ref) and actual position went to 0 ASAP. this made me feel like our controller was rendering a "fix" not a wall, given the wall constraint should be unilateral, a point-surface contact. Lei said I should only control the motion towards the wall a few times now but I never got it. what he meant was the "to-wall" direction error went to 0 and the rest left uncontrolled. in Q matrix, this means large gain in z coordinates and 0 in others, if wall is at `z = 0`. in my defense, this was the first time he explicitly said that.

this could work but it changes things. 

first, it means that our controller is more like a haptic renderer instead of a low level controller.
previously, I imagined the controller to work after collision detection.
the collision position was the ref signal and the controller kept the TCP there. no query about the environment was asked.
the "switching" of the controller/model was more about the dynamics of the device (it's slightly different in different regions if we pre-load the linearized model).
but for this new idea to work properly, the controller needs to know the surface normal. 
when collision is detected, the controller has to ask, what's the current surface normal for the collision.
then the corresponding/correct "controller" so to speak is used. 
this means the "controller" is based on the TCP position as well as the surface normal. 
the TCP position determines the control plant, the linearized dynamic model. 
the surface normal determines the Q matrix. 

second, this brings back the discussion why we did everything in joint space. 
we could choose either. but joint space was ideal for IJC. I originally thought joint space made more sense and was easier to compare the performance. 
and now that I think about it, it also had something to do with the states. 
the I/Os are important but we also wanted the states to be measurable. 
and our only sensor is the encoder at each joint.
this needs some exploration. maybe some derivation. 
however, for this new iteration of the controller to work, it's straight forward if everything is in task space from my current understanding. 
it could also be possible if we have a mapping to map everything back and forth.
that's kinematics, I know. but I'm not sure if it works like that.

third, this reminds me of virtual fixture/active constraints more and more. 
and on a side note maybe, this is more about the geometric property (surface normal) of the environment. 
we still consider everything stiff. 
if it works perfectly, no penetration.
but apparently that's not going to happen but we don't really talk about "stiffness" of things.
our stiffness is not a number. it's infinity. it should be. 
but maybe there is a relationship between the gains in Q matrix and the "stiffness". 
it's just that right now, we only consider the simple cases. 
very large gain means infinity stiffness wall.
0 gain means free space or at least no constraints.
the something in between is the thing that's missing.
circling back to virtual fixture/active constraint, it doesn't care about stiffness. 
it just keeps you away from (entering) a certain area. 
haptics rendering, on the other hand, renders a specific stiffness of a specific thing. 

## a quite random thought
when you do haptic rendering, on a point cloud level, is there really 6-DOF rendering?

I understand a 6-DOF device, b/c it could be a stick and a 2-point collision. 
but it seems that each point is a 3-DOF rendering and the end-result of that on the device is a 6-DOF one, with a torque.
to rephrase, if the basic unit of interaction is based on point-point interaction, there should be no torque rendering on that level.

# 20190731 - 0801
was I too deep into implementation that I forgot about research?
## reexamined the performance MIMOMOP vs. MIMO
last entry, I talked about, from a evaluation stand point, how things other than the controller itself could after the "score". most previous tests were run under `limTraX = 10`. I rearranged things and generated a new set with 25. this was chosen b/c 25 was reachable and could be currently considered the edge of the workspace. also, TCP could be far away from the origin (OP for MIMO) and close to outer OPs for MOP. this should be the area where MOP prevails.

I ran tests with offset to origin equaled to 25. MOP broke only at `y = - 25`. MIMO only held at `y = 25`. by "broke", I meant TAU went into singularity. this could be b/c the controller gain was "too soft". 

I also thought about adding limits to the joints. this was something on hold indefinitely. my simulation went into singularity all the time. first of all, it should not. secondly, this limits actually act as "dynamic constraints (exert force like a spring)" when reached. at least THIS implementation is not what we look for. it's also going to introduce forces that are never going to be there. instead, it should be a solver's problem,  not a constraint. it should work like a feasible range for a certain variable. when the solver solves the dynamic problem and if there are multiple solutions, the solver should pick the one that's "feasible". that's closer to the implementation we need. 

## misc
- used to change `strcmp()` to `strcmpi()` to be case insensitive. then realized it's still needed for choosing the controller and changed back. changed to number instead. 

- modified some of the names of the variables used in *genMultiOPs.m*, what to be saved, and save command in the script. apart from offsets and gains, control plant before controller design was also saved, considered as "static". in this way, when design new controllers, only design code was needed. control plant could be loaded directly.

- the params used in *multiOPsGS.m* were also saved from *genMultiOPs.m* so that they could match.

- added "fast mode" to run *exeScript.m*. when it's not the first time to run and the controller gain was already designed/loaded, no need to run those again. but failed. didn't really find a good logic to do this. manually for now.

- moved *plotAnimation.m* to EOL (stopDevel). it was for developing some plot animation similar to what we had in ADAMS simulation. the plot would change w.r.t time.

# 20190730
## further tests with MIMOMOP vs. MIMO
the overall conclusion still holds: the first is not superior to the latter from a simple look at the joint angle plots. and it should be no surprise that improvements vary from point to point. the closer to the OP, the better. where the OPs were for MOP and where the user pushed were important to the performance. for example, if we picked OP and pressing point both at the edge of the workspace, for MOP it would be perfect (right on OP) but for MIMO it would be the furthest away from OP. of course, in this case, MOP would be a lot better than MIMO. is this a good pick? how do we justify that? if we picked the center region, no doubt the two performed the same b/c the controllers were identical. maybe I should do a "grid query" of the whole space, or actually the whole wall surface, and then have some evaluation criteria to decide which is better and how much better it is. this probably means a lot of simulations. could borrow a better PC (more cores, or at least threads) to do this. 

to this end, maybe next step would be to change the *genMultiOPs.m* and generate gains and offsets with other params. there could also be some "hand shake" (passing the params through) between *genMultiOPs.m* and *multiOPsGS.m* either by save/load *.mat* file or straight up use the same variables. ~speaking of which, there was also a **TODO** related to separate "static" part from the controller design part from before.~

## ran MIMO simulation successfully in *disturbTest.slx* with *exeScript.m*
this was one step closer to "one script rules all". copied some lines over from *exeDisturbScript.m*. 

the plotting part of the script was moved out and saved as a new script, *pltScript.m*. this was definitely WIP b/c right now manual check was needed for everything to work.

## misc
- added frame viz representations (markers) for start pose (actually, position) to *disturbTest.slx*. to differentiate, the origin was green. the stop pose could be red but not implemented yet.

- in mechanics explorers, chose view/layout/four standard views to have a better look at the simulation from all angles.

# 20190729
## plot things with a script
with `sim()`, the data could be accessed as signal logging (enable logging) or output signal (with `Out` port). tried output but ended with signal logging. plot time series (a structure/class) directly. 

I was also thinking the usage of the *exeScript.m*. added prompt for user input in choosing the controller type with a default answer. it's possible to actually only have this script (*exeScript.m*) and this model (*disturbTest.slx*).

## solidified *playground.m* into *multiOPsGS.m*
the script was for multiple OPs gain scheduling purposes. the most important part was to initialize lookup tables and load in saved gains and offsets for GS. used to only have lookup table initialization for $X$. now added $Y$ and $Z$ but not used. in the simulink model, it's still only $X$. the simulation would work correctly as long as they were all "divided" in the same way. could be a **TODO** here.

## MIMOMOP work in progress (WIP)
currently, TCP position was fed back as the GS conditions. noted that the position from the simulation was in meters. also noted that most of the time I used millimeter (in lookup tables, design params. check the number to determine the unit. it should be quite obvious). 

currently, the gain was "switching" instead of "scheduling" b/c I hadn't found a way to properly run the simulation yet. but the "switching" seemed correct. the performance, at least from the joint angle plots, was some worse some better. this brought back the discussion how to evaluate. maybe by just looking at joint angle plots was not enough. from the animation, it seemed better. 

# 20190725
this was a "parallel" or "asynchronous" day in the sense that I had multiple "threads" and in the end I had to tie everything together.
## revisited *exeScript.m* for a clear script for *disturbTest.slx*
*exeScript.m* used to be an attempt to run simulation as a function and parameters were changed through different input arguments. that didn't work out. I settled with a script with some flags in the front. as mentioned yesterday and also previously, I also needed a good practice to do everything in a good order and clearly. I had scripts for previous simulation, this script now served as a initialization script for *disturbTest.slx*. there will be major revisions along the way but for now let's start somewhere. the lesson learned was that initial conditions for the simulation should be in the back, just before simulation, and in one "section" that I could quickly change and run just that.

this was also the time when I found out that I didn't do the separate of start, stop, and linop for disturbance rejection. previous implementations were for position control task, so to speak. moved back to *controlDemoLMwIAW1D.slx* to develop and verify this first and now it's done.

modified the reference signal for both MIMO/MIMOMOP controllers. additionally, for MIMOMOP, the ref signal was further changed. ref signal also needed joint angle offset compensation. in MIMO it was done in one `constant` block with an expression. in MIMOMOP I had to separate this and feed in the scheduled offset. this implementation was actually clearer in the way that it showed the ref signal also had to have some compensation. considered this an optional **TODO**.

## revisited all "exe" scripts for separate of 3 points
when I was rewriting *exeScript.m*, I realized `stopp_pose` was never used in the model. I thought it was due to copy-paste and I got some extra no-use code. in fact, I probably didn't really work on this for disturbance rejection. and b/c the initialization code was all over the place, I started to move things around.

turned out *exeDisturbScript.m* was the only one that needed changing. the missing code that used to be in *linScriptJS.m* was pasted in and the order was changed a little for everything to work. the 2 simulink models in the script were also updated based on their position control counterparts (mostly the ref signal part). now *exeDisturbScript.m* and *exeInitPosScript.m* could be run successfully directly. I imagined that all "exe" script should look more like *exeDisturbScript.m*

another thing I realized was that I didn't really do MOP for SISO. the "linearized model" for SISO was obtained by [locking joints and measuring/calculating moment of inertia](https://github.com/easyt0re/piecewiseLinearModel#20180822). it was quite a manual process so I didn't really further develop it to work with MOP. so for SISO/IJC, the `linop_pose` was always the origin. ~this could be an optional **TODO** but it's not that important currently.~

## the performance of MIMO in disturbance task
with *exeDisturbScript.m* working with different "position pressing the wall", I tried a few points. everything was OK for SISO. however, for MIMO: with offset along x axis, TAU would be pressed into singularity; with y offset, it was OK; with z offset, although it didn't make sense (the wall was `z = 0`), it also got initialized in singularity with positive offset. all the singularities seemed to be related to simulation. with the working cases, MIMO was worse than SISO. this went back to the fact that the current gains were obtained with position tests, which should give MIMO bad performance in disturb tests.

thus, **TODO:** 
- [ ] modification to *genMultiOPs.m* b/c controller design part should be the one part that could be modified and run many times, while the "static" part should be run only once and saved.

- [ ] find proper gains for disturb tests. maybe optional

- [ ] \(in general,) finalize the code currently in *playgound.m*. the lookup table was only implemented for `X` both in code and simulink

another thing was, I was talking about MIMO vs. SISO here in disturbance at different position. I didn't talk about or try MIMO vs. MIMOMOP. definitely **TODO** and one more thing remained was how to really "see" "scheduling" in the simulation.

## misc
`Goto` and `From` block in simulink had some "scope" issue. when in different levels, tag visibility of `Goto` block needed to be changed. and an additional `Goto Tag Visibility` block was needed for "scoped". but this didn't work. used "global" instead.

`Product` block had a "order" for ports when used to do "matrix multiplication". I checked the dimension of signals to make sure this was correct. otherwise, it wouldn't throw an error but do some weird reshaping instead if it's possible.

cut (commented out) initialization code in *linScriptJS.m* and pasted it in all "exe" scripts if needed, as a streamline step (hmm... I actually had it already in *exeInitPosScript.m*. I wondered why I didn't also do this with the others)

moved the initial condition code to just before running the simulation but `linop_pose` had to be before linearization.

# 20190724
## what's needed for a given OP
though the method is called gain scheduling, the things changing for different cases are more than just control gains. that's why *genMultiOPs.m* was rerun and the "offset" was saved. 

for each OP, we needed: control gains (`Ki` and `Kc`), position offset (joint angles at OP), velocity offset (which was 0 in this case), and torque offset (joint torques from the simulation at OP). all of these needed to be changed/scheduled for different OPs. although these were specified in `opspec`, they were not implemented in simulink models.

there was also an initialization conditions. 

these all went back to preparation before running *disturbTest.slx*. I should say that previous implementations from me were messy. "initialization code" was all over the place instead of being in one place and run before everything. this should be a **TODO** and, later, a practice. all "exe" scripts were subjects to changes.

## further development of MIMO MOP in *disturbTest.slx*
an MIMO MOP (multiple-OP) controller was added in the controller variant subsystem. the `Ki` and `Kc` gains were scheduled based on position of TCP. position of TCP was currently obtained directly because it's a simulation. otherwise, in real implementation, F.K. or other stuff should be used. **WIP**

## misc
- added *sdisp.m* for display messages with a flag

- added some printing commands in *genMultiOPs.m* to know the progress

- saved offset from *genMultiOPs.m* but it's actually no use. manually saved again

- tried to do `parfor` instead of `for` again and failed again

# 20190712
this was a bad day, idk. my brain stopped.
## built the module to look up control gains, sorta
quick recap: the implementation chosen for GS was "lookup table". the idea was to determine the "quadrant" the TCP was in and output the controller gain in that quadrant with that OP. but I didn't find out how to output matrix with a lookup table simulink block. since I didn't really find anything that would suit perfectly, I did it in a "manual" fashion. given a coordinate, I used 1-D lookup table (or `interp1()`) to determine the index for that dimension. then, I just output the set of gain with indexes for each dimension. there could be some bugs in the current implementation. I didn't take a closer look at all the numbers. 

one current problem was that, as previously implied, this gain look-up was based on the coordinate of TCP. however, I couldn't obtain that, not w/o F.K. the F.K. we had was an iterative one. not sure if it would work in this case. maybe it's also a good choice to drop the rotation DOFs at some point. it would make things easier and maybe make more sense. point contact could only have 3-DOF forces. torque is probably for multiple-point contact and that could be quite confusing for a position controller. again.

I added *playground.m* and *playgroundslx.slx* for simple testing and things. they should serve as temporary files and I would change them quite a lot. not in the git of things.

# 20190710
## section 3: change to desired states, augment, and design controller
surprisingly, this was done quite easily. the code was mostly from *linScriptJS.m*. at this point, everything saved was still 27 (or 27x1) instead of 3x3x3. I later realized that this was only the continuous time part. so this file was still unfinished. another thing I realized was that it's harder to debug `parfor`. whatever was wrong, the flag was always at the `parfor` line though the error was the real error. saved the designed controllers as *lqrGainwI_3x3x3.mat* b/c in the end, *genMultiOPs.m* should be run offline and the controllers should be loaded for real-time usage.

## the start of all controllers in one file
saved *controlDemoLMwIAW1D.slx* as *disturbTest.slx* and successfully wrapped the controller part into a variant subsystem. now, controller (MIMO/SISO, later continuous/discrete maybe) could be specified with a flag `CTRLTYPE`. tested with MIMO continuous time and it's working. need to do the same for the rest. the result of each run could be saved in a `SimulationOutput` structure for plots later. 

many more things could be done but I should follow one through. the rest could be copy-and-paste then.
- [x] copy paste all kinds of controller here (no rush, multiple OP first)

- [x] maybe "disturb" and "position" could also be in one file

- [x] maybe continuous and discrete controller should only have differences in values but the structures the same, meaning switching scripts for calculation instead of switching controller subsystems.

- [x] maybe rewrite how things are loaded and saved, including plotting scripts. eventually, all the "exe" scripts would be rewritten.

- [x] think about how to do the simulation for multiple OPs

# 20190709
## speed things up with parallel stuff
for *genMultiOPs.m*, the outer for-loop for OPs could be [`parfor`](https://se.mathworks.com/help/parallel-computing/parfor.html). there's also [`parsim`](https://se.mathworks.com/help/simulink/ug/example-of-parallel-simulations-workflow.html) to run simulation in parallel. check the [function page](https://se.mathworks.com/help/simulink/slref/parsim.html) for more info. these could be used in other processes as well, like *lqrSweep.m*.

## section 1: calculate motor torque compensation
so, it's tested that you cannot use `sim()` in `parfor`. `parsim` was suggested. section 1 was done with `parsim`. my work laptop could only have 2 things (workers) in parallel. to use`parsim`, define `SimulationInput` for each simulation first, then, run the simulation using `parsim` with the stacked "options".

here again, I ran into the problem of which workspace could be accessible during a `parsim` call. it seemed that `parsim` couldn't read global workspace that's populated by the "main level" scripts. and I learned to use DataDictionary (\*.sldd files). I ran *simTAUInit.m* which contained all the "design params" for TAU and saved everything in *TAUdesign.sldd*. I linked it to *simTAUcheckOTorq.slx* (used in section 1 simulation) and *tauControlPlant.slx* (used in section 2 to generate linearizing options instead of *simTAUJTDlinmod12.slx* b/c I didn't want to modify the latter file). after this, somehow, `parsim` could see these values. it makes sense actually. for things to work in parallel, you have to "clone" workspace and you have to specify it. 

## section 2: specify linearization option
tried to write this part with `parfor` but failed. ran into the "accessible workspace" problem again and tried to solve it with linking DataDict to *tauControlPlant.slx*. went past this problem but yielded an error I couldn't find the solution. it's still related to this `parfor` implementation. currently, I went back to normal for-loop instead. maybe I should get rid of the inside for-loop. it's just a way to have less lines of code. 

other than that, section 1 and 2 could run and finish w/o error. I couldn't verify if the values were correct. 

# 20190708
the tense of this log is always a pain to me
## rethink the IOs of the TAU model
this is actually from yesterday. through yesterday's learning, at one point, I felt like what they did in the tutorial was to define disturbance as input but can be "turned off" as "uncontrollable" input. in our case, we could define the F/T on TCP "unmeasured disturbances" with white noise by default and the joint torque "manipulated variables" if I understand it correctly. like I said, there are too many options and data structures I don't know in MATLAB. then the "input" is rather general, as in "all the signals that go into the system" and specifically defined later for different analysis, like disturbance rejection. 

## further study of LPV
after reading examples of LPV, it seemed that it's good to approximate the plant. it's basically a stack of state-space models and based on a condition, it would switch among these models. my current understanding was that it would be hard to implement controllers this way. anti-windup and saturation made it not a simple ss model. so the current idea was to go back to GS and GS in this case was just gain look-up tables with a condition. not too much would change in the structure or the implementation of the controller. still, these example codes were good examples to implement things. 

## modified *simTAUcheckOTorq.slx* only to hide animation
all this time, the sole purpose of this simulation model was to calculate the motor torque to keep the mechanism at a certain pose, the compensation for OP in motor torque. showing the animation was unnecessary and from now on, I might use this in for-loop for multiple OPs.

there could be some "interfacing" problems with all my codes and models. here, in this file specifically, in each joint, the position state target was `init_joint(i)`, which was the initial value I think, and for each joint, the input position signal was `runtime_joints(i)`. they all had "sign-flip" in the model, so no flip was needed.

## added *genMultiOPs.m* to generate things for multiple OPs
at this point, the purpose of this script was undefined. it could be to generate control plant (for later control design) or controller directly. this was the first version of the code and it seemed working. currently, this only divided in position, not orientation. the extension should be easy. 

this was still the model with auto-generated states. ~next steps should be, as before, change to desired states, augment states, design controller.~ this should be done in a parallel way to speed up instead of a for-loop. should be doable. 

# 20190707
## revisited *simTAUJTDlinmod12.slx*
*simTAUJTDlinmod12.slx* was the control plant model. it had the I/O setup and nothing else. it's also why, in linearization, this file was used. saved a copy and named it *tauControlPlant.slx* to test new things on it and left the old file untouched. I followed the video about [model trimming and linearization](https://se.mathworks.com/videos/trim-linearization-and-control-design-for-an-aircraft-68880.html) but all the functions mentioned seemed to be already implemented in the code. the tutorial was more of a GUI tool. and also the last part of the video was only good for SISO. so the development on *tauControlPlant.slx* was stopped. but I didn't remove it. maybe I will circle back to it and at least this could be a copy of the IO system

## much to read and learn from MATLAB main page
I started with the "wrong" keyword - ["adaptive MPC"](https://se.mathworks.com/help/mpc/ug/gain-scheduling-mpc-control-of-nonlinear-chemical-reactor.html) and gain-scheduled MPC was also kinda adaptive I guess. apparently, there would be 4 branches to solve this. the first split was whether or not a linear plant model could be obtained online. if yes, successive linearization or online model estimation could be used with adaptive MPC. if no, you could do gain scheduling (GS) or linear parameter varying (LPV). maybe the words here were not accurate but everything was on the page. 

I assumed that, for our case, linear plant couldn't be obtained online. the being said, there was no specific reason to use MPC. I was only looking for a method to switch controllers when needed. still, MPC could be interesting as I noted in my previous discussions with other people. Lei might have said something about dSPACE cannot do MPC. 

at this point, [GS](https://se.mathworks.com/help/control/gain-scheduled-controller-tuning.html) and [LPV](https://se.mathworks.com/help/control/ug/linear-parameter-varying-models.html) both looked promising. the only difference that mattered to me would be the ease of implementation. 

along this path, I actually found out how to do different controllers in the same *.slx* file. the block is called "variant subsystem". it has a "flag" to switch from one to another. it could be useful to reduce the number of the files. I could switch controllers as well as models (position/disturbance).

just too many things I don't know about MATLAB and SIMULINK. also found out about this (signal) selector block. currently I have no use for it but I think it could be useful. 

# 20190701
## generated some animations
for the ICCA presentation, Lei suggested to have some animation. I generated animation with the Simscape 3D model. currently, the motion was too small to observe. I tried to also animate the plots. that didn't work out and I gave up. I wanted to do something like plots in ADAMS, synced and changing with the simulation. I also learned how to record screen or a window in Win10. 

added frame viz representations (markers) to *controlDemoIJ1D.slx* and *controlDemoLMwIAW1D.slx*. this was for the animation to show the deviation at TCP clearer. for the marker, the ball was with a radius of 2 mm and the axis was with a radius of 1 mm and a length of 10 mm, just for reference.

## again, note the setup for the simulations
for ICCA paper, `posGain, errGain = (1e6, 1e9)` for MIMO and `Trise = 0.05; Mp = 2/100;` for SISO. the disturbance was 5 N and the simulation was 0.5 s. this resulted in the plots in the paper and the deviation caused by the disturbance was not that obvious, meaning it's hard to tell from the simulation animation. we could try with the same params but 20 N and see what would happen.

for discrete time extension, because of the encoder resolution, a new set of params needed to be found. as logged previously, `posGain, errGain = (1e3, 1e7)` for MIMO and `Trise = 0.1; Mp = 5/100;` for SISO. the disturbance was 20 N and the simulation was 1 s. this yielded some proper deviation for the encoders to detect. 

sometimes it's quite confusing and tedious to do these book keepings. this was why a better structure of the code was needed.

# 20190613
## modified *controlDemoIJ.slx* reference signal part
I separated start, stop, and linop [before](https://github.com/easyt0re/piecewiseLinearModel#established-start-stop-and-linop-separately) for LQR controller but I didn't do it for IJC b/c IJC was having problems maybe. should have left a TODO flag. anyway, this was done now. not sure if this would work 100% but it might. I had problems wrapping my head around this "linearized" PID controller last time and this time still.

the reference signal was done better with one step using the vector instead of 6 steps and a mux. and the meaning was clearer. it seemed that the reference signal should always be from zero to desired position, which was actually not an absolute displacement but a difference, the difference between target and initial position.

(20190725) a part of this "separate" (the 3 points) code was also copy-pasted into *exeDisturbScript.m*. but I probably didn't really work on the simulink model to make this separation work. 

# 20190612
something is always missing here. so the point of the log is only to have less missing things.
## tried to structure the code a bit more
started *exeScript.m* from *exeInitPosScript.m* and failed

tried to do it like Python `python script.py --option=some_options` but didn't find MATLAB's support for this. tried to do it like a MATLAB function with optional input arguments. this was also unsuccessful b/c there were some problems with which workspace (scope, function call and level of workspaces) to use when call `sim()`. there was an option `'SrcWorkspace','current'` that I could use but then the question became where were the numbers that were "saved to workspace"? this could be revisited but not the focus of the current study. 

## tried to sweep for good Q matrix again
for some reason, previous logs didn't have everything I wished to record. there was supposed to be one set of weights for position control and another for disturbance rejection. the set for disturbance rejection was not explicitly recorded in the log but I guess it's the one we were using this whole time: `posGain, errGain = (1e6, 1e9)`

this time it was for a different reason: we did the discretization of the system and I should be writing about this in a new journal paper. previously, with continuous time, I didn't implement encoder (quantization). so with the previous weights, the error was too small (too good) for the encoder to pick up. I might have a note on this, for previous simulation results, it was too good. there was no observable movement on the TCP when it was disturbed by a force. basically, it's not realistic. to achieve reasonable and good performance, I need to find a set of weights that, under a disturbance (now it's 20 N, previously it's 5 N), yielded a deviation of 5 times the resolution (0.0077). so maybe something around 0.01 rad. and it also had to recover fast. 

these sets were fine with me: `posGain, errGain = (1e3, 1e6), (1e3, 1e7), (1e4, 1e6), (1e4, 1e7)`, but I chose `posGain, errGain = (1e3, 1e7)`. I also did a "finer" sweep within this 2x2 region but didn't find anything interesting. with these weights, it showed some "proper" performance that we could work with.

# 20190609
## on-going problems and thoughts
the oscillation of the control input should be eliminated and could be caused by these things:
- the encoder reading was quantized, meaning the error could never be zero, hence, the control input. in this case, we could **either** quantize the reference signal **or** make error less than a resolution zero

- a closer look showed that joint 6 was saturated every now and then. and the target position for joint 6 was not reached in the end, not because of the quantization. was this oscillation in control input similar to that in IJC (see previous logs)? the obvious difference was that this oscillation was quite small and around the correct point. but still, this could be because coordination was bad due to discretization. this could be caused by competing signals in MIMO system

## tryouts and decisions
- put "dead-zone" on error
	- the idea was that, when the error is less than 1 resolution, error should be set to zero. a similar function was dead-zone but not really. I ended up writing my own MATLAB Fcn. maybe I did something wrong but, in general, the control performance was better without this module. it could be because it introduced some delay and that's bad. as of now, this was **stopped**.

- quantized reference signal
	- since the encoder was quantized, I could also quantize reference signal. this introduced some "error" for the controller because the signal before quantization was desired, ideal, while the signal after quantization was "as best as it could get". the before was also a valid solution of IK while the after was probably not. the before was the real OP and the after was a bit away. 
	- however, after this, `e = r - y` should only be any integer times resolution, which would eventually go to zero.
	- this was tested out to be a good thing to have. the system would never reach desired pose but that's inevitable because of the encoder.

- increased saturation limits on motors
	- this seemed to be a must now and at least began to solve most of the problem. changing `Vmax` from 2 to 5 yielded smaller oscillation in control input. and for joint 6, it was just oscillation between the upper and lower bound. from the position output, 5 gave a more steady position at target while 2 gave some oscillation between the target and one notch below (observed in joint 6). combining input and output together, with 2, it seemed that joint 6 was doing "reach but cannot keep". 
	- like I said, with higher saturation limit, things started to improve a little. but it didn't solve it all. there were still small changes in position output as well as in control input. this could be because my controller was too sensitive to noise and although the current controller was a centralized design, the I/Os were still competing with each other.
	- obviously, there was a problem in doing this. it cannot be done in the real system. or I have to find a way to justify that we could have this. I was thinking about introducing gear ratios but it would definitely cause more trouble than solve.

## discussion with Yuchao and Yu
- a closer look with the transfer functions might be helpful to figure out: frequency domain performance, disturbance rejection, positive/negative effects from Is to Os. there would be 6x6 tfs and maybe process them with `zpk(), minreal(), dcgain()`
- moving poles "blindly" was not as efficient as in SISO design. and sampling frequency should be 10 to 30 times faster than the fastest pole was a recommendation, a rule of thumb, out of experience. it's OK if it's violated. 
- a simple thing to do was actually check the stability of the discrete time system. if it's stable, then the sampling time was fine. 
- current state vector was `[q_dot; q; integral_e]`. this could actually be converted to all e-related expressions. it might give a clearer representation and maybe helpful in solving some other future problem (e.g. the reference signal in current problem was a constant instead of a variable). 
- now the problem was solved with LQR, maybe close to an MPC without constraints with infinite horizon. to continue this path, it could be changed into an MPC with finite horizon and with constraints. 
- to further this thought, error could be done as an MPC constraint rather than an extra state in the state vector. currently, vanilla LQR had steady-state error, so we "augmented" the state vector with the last bit, adding an integration on error and we hope to control all states. with MPC, maybe we could keep the previous state vector with only velocity and position, then eliminate error with a constraint. 
- even with vanilla LQR, I didn't really push it to full potential. there were many DOFs but I only used 3,4 numbers to parameterize the controller. H-infinity is another control idea that could be useful in our case. 
- maybe all this time, I was, again, emphasizing on the wrong task. so far, I tried to do position control with the system. but the ultimate goal was probably disturbance rejection. these two controllers might be different, at least at the level of different weight matrices. this was first shown in [this log](https://github.com/easyt0re/piecewiseLinearModel#tested-with-disturbance) but I didn't think about it too much. in the paper, I only reported the disturbance case with the `[1; 1e6; 1e9]` weight. if I remember it correctly (and ironically enough, this was not documented, after all these logs), this weight was good for disturbance rejection and too high for position control. this could also be why there was jitter when doing position control. maybe it would do better with disturbance. 

# 20190530
## discrete time again with LQRwIAW
the file was saved from *controlDemoLMwIAW.slx* to *DcontrolDemoLMwIAW.slx* and the old file *DcontrolDemoLMwI.slx* was moved to EOL.

the controller design, as before, was done in *lqrDisc.m*. `lqrd()` was used instead of `lqr()` for continuous time. Note that there is also a `dlqr()` but it's for discrete plant. our plant for now was continuous. 

(20190609 as it turned out, in `lqrd()`, `dlqr()` was called probably. so `lqrd()` basically do continuous to discrete first, then called `dlqr()`)

zero-order hold and quantizer were implemented and, surprisingly, not too much performance drop occurred. the computed control input for some reason was oscillating all the time. as a result, all joints were moving all the time. this could be something to look into next.

I wonder if I should also put quantizer in the continuous time model. it seems that quantizer doesn't make the system discrete. time can be continuous but there is a smallest difference for the encoder.

# 20190528
## some findings and understandings before moving poles
so far, I've been struggle to understand the concept of poles for MIMO

there was actually not too much to say. at one point I thought my way of calculating closed-loop poles for MIMO was wrong. I then used `pole()` and verified that `eig(A - B*K)` was correct, at least in this case.

following Yuchao's suggestion, I also tried `ss2tf()` on my model. it's confirmed that it's indeed an MIMO system. it should be noted that state dimension has little to do with whether the system was MIMO. it's the number of I/Os that matters, as the name implies. and it's also important that they are coupled, interacting. this was also confirmed. otherwise, it's just a bunch of SISO systems written in a matrix form. it can be easily divided into several SISO systems and solved. 

so what did we learn? after the LQR, some of the closed-loop poles are, indeed, quite fast. it could be that my gains/weights in Q matrix were too high but is it possible to move the fast poles to slow w/o losing performance? I think that pole was there for a reason and though that pole was fast, sampling time was not a problem when LQR controller was moved to discrete time with a reasonable sampling time (see [this log](https://github.com/easyt0re/piecewiseLinearModel#added-discrete-time-lqr-lqrdiscm)). I think high gain/fast pole was the consequence of the rise time requirement. that's why it shouldn't be moved.

after reading the log mentioned above, I was a bit confused about the claim that "closed-loop poles were slower than 120 and reasonable". was this a time when the gains/weights were different or for a different test/task (I had different Q matrix for force disturb and position step)?

# 20190514
## established start, stop, and linop separately
everything was done with *exeInitPosScript.m* and *controlDemoLMwIAW.slx*

tested with linop is/isn't origin and start/stop different poses

added frame viz representations (markers) at origin and TCP. the one at origin (not moving) was darker

test log: `linop = [t_x, t_y, t_z, r_z, r_y, r_x];` 

- to have non-zeros in all joints, we changed `r_z`. with everything else sets to 0, `r_z` could range from - 0.1 to + 0.3 with step of 0.1. - 0.2 and + 0.4 led to strange IK solutions. + 0.5 broke the assembly. 0.3 (rad) is around 17 degree

- after a closer look, it's not the problem of IK but of simulation. if TAU was initiated at, say - 0.3, directly, TAU went into "singularity". however, if it initiated somewhere else and moved to - 0.3, everything was fine, proving IK is correct. 
**TODO:** more reliable initiate process or some limits on passive (spherical) joints. initiation is used in static torque computation and start the simulation

# 20190513
although I tried to keep track of everything, the project is still a mess

the scripts and models are getting more and more and they are poorly managed

we have:

- two tasks: step in position, F/T disturbance at TCP

- two time domain: continuous, discrete

- two controller design: PID, LQR

## incomplete work in continuous time
- [x] LQR saturation and anti-windup implementation

- [ ] LQR with smaller gain using pole placement

- [x] start and stop at arbitrary places regardless of the linearized OP

- [ ] joint range limits to help with simulation/animation (20190525, see [log 20190514](https://github.com/easyt0re/piecewiseLinearModel#20190514))

according to this, maybe I should drop development in discrete time for a while

## incomplete work in discrete time
- [ ] stable behavior for both controllers in both tasks

- [x] quantization in encoders and check other aspects of discrete design

## change the "structure" of the code
- [ ] implement things as functions instead of scripts with no input args

- [ ] a [Simulink Project](https://se.mathworks.com/help/simulink/ug/create-a-new-project-from-a-folder.html) can be created for better structure of the files maybe. now it's already a mess. see more tutorials [here](https://se.mathworks.com/products/simulink/projects.html). and another blog for setting folders for [Simulink cache and generated code](https://blogs.mathworks.com/simulink/2015/01/16/controlling-the-location-of-the-generated-code-and-temporary-files/) but this is basically using the idea of Project.

## added AW to continuous time LQR
previously, only saturation was implemented with no anti-windup (AW) in *controlDemoLMwI.slx*. I tested some ideas before but failed. the problem was probably b/c the order of integral and gain on the "I" path.

implemented and tested a new version and it's working. with initPos test, the position is smoother. saved as *controlDemoLMwIAW.slx*. copied the controller part to *controlDemoLMwI1D.slx* and saved as *controlDemoLMwIAW1D.slx*.

the difference between *controlDemoLMwI.slx* and *controlDemoLMwI1D.slx* was not so clear at this point. I checked but didn't find why I had 2 models for 2 task, separately. obviously, the disturbance part was done differently but I don't remember the reason. the development of these 2 models would probably be stopped.

# 20190403
## OK, I didn't fully understand what Lei said, again
first of all, this position controller was only used for stiff interaction

so to render a spring, some other controller or algorithms should be used

secondly, "writing on the wall" case can be done with a computed reference signal (not sensor feedback)

when there was a collision, the reference position is always a point on the wall, so it's different from the encoder reading

this could work actually although it could be difficult to implement

and maybe the control burden was shifted to those conditions: instead of a complex controller, you have conditions to evaluate

now I would say it sounds really promising. maybe I should push this through

they had no more further comments on the stuff Kjell brought up

# 20190402
## Kjell mentioned about the size of the "step"
this was quite interesting and practical yet i never thought about this

since the rise time is 50 ms, we basically asked the system to cover the step within this time

if the step was 5 mm (the number I used in previous tests), the velocity of the TCP is 5 mm/50 ms, which is 0.1 m/s. is this reasonable?

and to check the acceleration in this logic, it's probably 8 m/s^2. is this reasonable?

and according to the paper I read recently, there is definitely some max performance (i.e. acceleration at TCP) that can be computed

and also, maybe it is the haptic control algorithm's duty to keep the actuators from saturation

for the second point, my question is, how? could this be my next mission? at least it sounds challenging and complex

## thoughts on Lei's direction
I was really confused about how I could place his vision under the scope of haptics

what he wanted me to implement seemed something that had never been done before

to me, this could only mean, a) I didn't find the papers yet, b) this was somehow proved impossible, wrong, or too hard to do

I always thought something was wrong about this but let me just have faith for a little while

we proposed to use position control to solve haptic rendering, which was mostly done as open-loop force control

so this idea couldn't fit in the figure of a "traditional haptic rendering" loop or structure

I think this position controller idea only works on stiff object with infinite stiffness

in other words, I don't think this can render a spring

consider a k=1 spring with user pressing 1 inward, how do we render this with a position controller?

the position should not change if the user doesn't move but we cannot know that

and the user should feel force=1 pushing back. we cannot do that with position control

at first, I thought it was admittance control instead of impedance control but it was also not the case

and I think admittance control works only with a force sensor. we have a force sensor but it's another story

SO, maybe this cannot render a spring. so what? is stiff rendering not good enough for you?

OK, let's take a closer look at stiff rendering.

let's just assume that the user wants to move on a stiff wall. in this case, the controller should try to keep the TCP stay where it is at each time step

but it seems to me that the reference signal is actually position feedback from the encoder. would this work?

then how will I or the controller know if TCP reaches its goal? and actually reference-feedback will probably be 0 all the time

# 20190312
## too many changes and it worked
contradict to 20190308, `A0` should be slower for this to work

but this was just the result. the reason behind could be that faster `A0` made faster system, hence the sampling time should be faster to have stability

this was something I didn't know how to solve

chose 0.9 as the damping and it seemed working (this was w/o calculation)

assigned different rising time for all joints. this didn't solve the problem entirely. it's clear that joint 2, 4, and 5 took longer to settle but since they didn't move to much, it didn't really matter

check for closed-loop poles for continuous/discrete time were added and they seemed OK to me

and it should be noted that *indeJointControlDisc.m* was quite different from *indeJointControlCont.m*

# 20190308
## had a meeting with Binbin
tried to refresh my memory with dynamics through virtual work

current understanding was: it's based on $Fv=0$ and $F$ is acceleration force

the assumption of 0-mass link was also discussed

it could simplify things but Binbin was not a big fan of it

she suggested another simplification so that all legs are identical in kinematics

## had a meeting with Lei
demonstrated most findings I had with all the discrete time simulations and learned a few things

`A0` should be faster rather than slower b/c it seemed that it's a model error rather than sensor noise

different rise time seemed to have better response in PID. fast: `6 > 1, 3 > 2, 4, 5`

could play with damping and `A0` a bit

should have `pzmap()` some where in discrete time design to check zeros/poles in disc/cont close/open loop (this was mostly for stability margin and stability check)

it's hard to tell why there was a high freq vibration

for LQR, I could have an anti-windup at the integral

for LQR, I should try to understand why such high gain and if possible, pole-placement high speed poles to low

# 20190303
## played with A0 in discrete time
Lei mentioned the reason of oscillation was maybe sensitivity to model error not sampling time

previously, it was always `A0 == Am`, b/c it seemed that faster A0 required even shorter sampling time

and previous results weren't wrong: when `samplingTime = 0.1 ms`, all problems went away

it's just that this time scale was not reasonable and maybe we could do this with a reasonable sampling time

let's recap and look at the result differently

first of all, it's still individual IJC only and step ref was calculate from `init_TCP_pose_sim = [0; 0; 5; 0; 0; 0];`

currently, I wasn't totally sure why this offset influenced the performance

it could be that larger offset made it easier to go to saturation

with 1 ms sampling time, `A0scale = 5` seemed to be quite good

larger than 5 didn't do much; less than 5 had slower response and nonlinear behavior

on a second thought, would this "5" be associated with the offset, meaning there could be a different "optimal" scale for a different offset

it should also be noted that the sampling time was not that short anymore

with fast observer part, the nonlinear behavior in position went away but the control input still hit saturation

after all of this, individual IJC seemed working but *exeInitPosScript.m* was still not working

# 20190228
## revisited the notion of moving everything to discrete time
~(it should be noted that the saturation for LQR controller in continuous time was not implemented)~
- [x] controller design in discrete time (*lqrDisc.m*, *indeJointControlDisc.m*)
- [x] zero order hold in Simulink model (*DcontrolDemoIJ.slx*)
- [x] quantization of sensors in Simulink model

## played around with sampling time again
this was done with individual IJC only, no interactions among them

`freq_0 == freq_m = 100; freq_sample = [1000, 3000]; samplingTime = [0.33, 1] ms`

the above was some calculation and usual recommendations (rule of thumb)

however, there was an obvious nonlinear behavior if `samplingTime > 0.3 ms`

accompanied with this nonlinear behavior, oscillation within the saturation limits was also observed and assumed to be the reason

if this was not "perfectly" working, one couldn't expect this would work when there was additional interaction/disturbance

and when moved this to *exeInitPosScript.m*, it worked when `samplingTime = 0.1 ms`

## added saturation to *controlDemoLMwI.slx* for LQR controller
the performance was a little bit worse but it was OK

the sampling time could be 1 ms

signals before/after saturation were logged but only before was saved to MATLAB workspace and plotted

same saturation was also added to *controlDemoLMwI1D.slx*

after this, a zero order hold was added and the model was saved as *DcontrolDemoLMwI.slx*

more things were needed for *DcontrolDemoLMwI.slx*

# 20190105
## extended the test to even tune gain for velocity
after discussion with Tong, tried to fix `errGain` and grid search the other 2

this didn't work out. for unknown reasons, it stuck the first time `velGain` was large

I think I need to take a step back and think about the paper and what is all of this

# 20190103
## tested with disturbance
previous tests were done with initial position script and the params chosen did OK

the problem was after doing disturbance tests, the new system could not keep up (new system had lower gain compared to previous tuning)

ran the same 10x10 tests with disturbance simulation, by first look results showed that `errGain >= 1e6` seemed to be comparable with PID

turned off the visualization for simulations from the settings for *controlDemoLMwI1D.slx* and *controlDemoLMwI.slx* b/c had to run multiple tests and that slowed down the process

**TODO:** would be nice if this could be done in script

**OK, I got EVERYTHING WRONG!!!**
## coarse search for LQR gain, AGAIN!
I flipped the axis and Fed up some other things, so all previous "conclusions" were all wrong

if the plots were correct this time, conclusions:
- with the same `posGain`, higher `errGain` meant shorter rise/settling time but also larger overshoot
- after `posGain > 1e3`, LQR would always be faster than PID regardless of `errGain`
- after `posGain > 1e3`, there would always be an overshoot regardless of `errGain`
- after `posGain > 1e6`, there weren't much difference when time scale was 0~0.2 s, it became a very fast system with very small overshoot
- with the same `errGain`, higher `posGain` meant shorter rise/settling time, and probably smaller overshoot, but it also seemed that this shrinking overshoot effect was not very obvious when `posGain` was too small compared to `errGain`
- after `errGain > 1e5`, LQR would always be faster than PID regardless of `posGain`

the above was observed from joint 6, the trends seemed universal but the threshold could be different for different joints

the acceptable params could be: `posGain, errGain = (1e3, 1e4), (1e3, 1e5), (1e4, 1e4), (1e4, 1e5)` 

# 20190102 Happy New Year!
## coarse search for LQR gain
added *lqrSweep.m* for the real sweep and save data to workspace of MATLAB

this would take a while to run b/c of simulation for the first time

added *lqrSwpPlot.m* for later plotting straightly from previously saved data

this added a "reference line" (horizontal line) to the plot

the scale of y axis was currently set to all the same, could be very coarse for some joints, but it also reflected the fact that some joints were "insignificant"

taken a first look, the conclusions were: 
- higher position gain had shorter rise time, larger overshoot
- higher error gain had less overshoot, faster recovery from overshoot, and a bit shorter rise time

the unrealistic gain could be due to unrealistic requirement

the ultimate question would be what haptic refresh rate at 1kHz means to a position controller

after a closer look, `1e3, 1e3` seemed a set of good params for comparable result with PID (rise time and overshoot)

the current problem was that settling time was too long

on the other hand, the close-loop poles calculated by me didn't change too much given all different gains

**TODO:** ~how to calculate poles for MIMO or~ what does my calculation mean?

plotted 10x10 gain from `1e0` to `1e9`, `errGain = 1e3` seemed proper for the rise time, similar or slower than PID, `posGain = 1e0 ~ 1e5` seemed OK. this was mostly with "undershoot" and the settling time was way to long compared to PID

# 20181227
## more tests with reasonable LQR gains
Lei finally suggested that the gains were unacceptable

unsolved problem: nfreq for PID was 100 rad/s but poles for LQR were at 10^3

argument for this: but they have actually comparable performance with *exeInitPosScript.m*

Lei commented: the difference should be around 10, 10^3 was too much

so more simulation results were needed to tune the gains

simulation log (simulation time = 10 s, R = 1, with *exeInitPosScript.m*):

- 1:10:100, after 3 s, everything was settled, overshoot could be 10%
- 1:10:10, overshoot was smaller, the rise time was longer, very slow to settle
- 1:1:10, compared to $1^{st}$ it's slower, larger overshoot; compare to $2^{nd}$ it's slower, larger overshoot, maybe faster at settling
- 1:10:1, compared to $2^{nd}$ it's a bit slower, less overshoot, slow to recover after overshoot, very slow to settle, over 10 s
- 1\:100\:1, overshoot was even smaller but the recovery from the overshoot was too slow, the rise time was quite fast, after 0.5 s it's almost at the ref with a bit overshoot

lost in all the plots, started to do sweep search

# 20181201
## tried zero order hold but didn't improve anything (discrete)
added 3 zero order holds on input, output, and feedback of the IJC controller and saved as *DcontrolDemoIJ.slx* based on *controlDemoIJ.slx*

it had perhaps the identical performance

tried 1 ms and 0.1 ms, no noticeable difference w/ or w/o zero order hold

and with 0.05 ms, the system was back to normal

# 20181120
## ran more tests to tune IJC discrete params
since PID was not really tuned but designed, the idea was to play around with sampling time, saturation limit, and maybe A0 freq

- design params were fixed with `Trise = 0.05; Mp = 2/100;`
- `freq_m = 100; freq_0 = 200; freq_sample = 1000`
- current setup was sampling 1 ms, saturation 2 Nm, A0 = 2 Am, and for offset test it couldn't get back and start oscillation which seemed to be due to saturation. this was also noted previously. disturbance test seemed to have similar results. it seemed that after discretization, the motors were not as powerful
- having shorter sampling time didn't really help (0.5 ms, 0.25 ms). no matter what I do, the system seemed to go into a high freq oscillation
- increasing saturation didn't help and actually allowed the system to go into singularity. more than 500 (2000) was needed to have 0 saturation but that's unreasonable and instable. 
- just in case I made mistakes in A0 freq calculation, A0 was set to Am again. not helping
- so, ok, shortening sampling time *DID* work. 0.1 ms got it stable with A0 = Am. sampling time was basically 100 times faster. maybe at this point it's fast enough to be considered as continuous. maybe it didn't work before b/c I used to have A0 = 2 * Am

to sum up, I suspected that the motors were acting against each other, otherwise, there was no need for huge control input signal

## added discrete time LQR *lqrDisc.m*
this was done simply by `lqrd()` in MATLAB

it's more or less the same with `lqr()` except for an extra arg for sampling time

it's quite amazing to see the system was still stable even if the sampling time was 0.01 s albeit the system was too slow after the frustration from what happened with PID

0.1 ms definitely worked and 1 ms showed a deteriorated performance but still OK (better than IJC)

added one line in the back to check poles of close-loop system and it's slower than 120, which was reasonable

# 20181116
## added scripts for System Objects to draw figures
used *visualDemo.slx* to draw Simulink diagram figures for the paper

system object was used to have proper name displayed at the center of the block

it was definitely an overkill but I hope this function could grow into something useful for MATLAB

## ran more tests to tune LQR params
I should have logged all the tests I ran and why I ended up with these params

IJC params were fixed with `Trise = 0.05; Mp = 2/100;`

IJC performance: with individual joint, it's what is designed; with TAU, it's similar, the signal settled between 0.1 to 0.2 s

for IJC and LQR, the joints with close-to-zero movements had unpredictable motions or some small oscillations

- with `Q = [1, 1e3, 1e6], R = 1`, overshoot was a lot larger than IJC, rise time was short than IJC but settling time is between 0.2 and 0.3 s; torque wise, IJC saw larger amp and high freq oscillation while LQR was smoother but joint 6 reached saturation (torque limit = 2 Nm)

- with `Q = [1, 1e3, 1e6 (1e9)], R = 0.1`, overshoot was even larger in position but the overshoot disappeared quite fast, there was saturation in motors, 1e6 and 1e9 made no difference

- with `Q = [1, 1e6, 1e9], R = 1`, this was the set of params I went with in the paper, the overshoot was smaller than previous tests but still larger than IJC

these tests above were done with a initial pose other than the origin

it should have a different response to disturbance rejection tests

- with `Q = [1, 1e6, 1e9], R = 0.1`, the deviation was smaller and the input torque was faster and a bit smaller actually

- with `Q = [1, 1e3, 1e6], R = 1 or 0.1`, the position was too slow so there was large deviation, the torque was also slow so maybe it's a bad set of params

- `R = 1` vs `R = 0.1`: in offset test, 0.1 had larger input with less overshoot, the rise time was also shorter, these were supposedly because of allowing larger input; this change didn't have much difference in disturbance test

after these tests, the params should be `Q = [1, 1e6, 1e9], R = 0.1` with a good performance in disturbance rejection. and since 0.1 and 1 didn't make much difference, 1 was used for R to punish a bit input

Note that saturation of LQR controller was not address currently

# 20181114
not much is done, I just want to log it for future reference
## started to try discrete time control
short answer: it is not working

tried to have shorter sampling time (0.0001 s) and rise time (0.005 s) but that wasn't feasible because the motors were saturated all the time

currently it was set to 0.001 s and 0.05 s and individual motors seemed to work

set `A0 = 2 * Am` to be consistent with continuous time

with a `step = 1`, it was not working due to saturation also

we should avoid saturation because it basically means the motors are not strong enough

one simple solution would be having larger saturation threshold

individual motor control exhibited similar plots but discrete time controller couldn't even hold the TCP at origin, where continuous time controller had no problem

discrete time controller would deviate and oscillate for quite long with no sign of stabilization

it seemed to be because of the saturation as well

it should note that previously the motor were already operated on the edge of saturation

renamed *indeJointControl.m* as *indeJointControlCont.m* for continuous time and saved as *indeJointControlDisc.m* for discrete time

## struggled with keeping track and tried to push this into KTH Github
fingers crossed that it would work

# 20181108
## added post-processing in *plotControlDesign.m*
generated a table from the figures for maximal deviation and torque

## modified all the joint angles to vary around 0
basically, redefined the zero point of all joints so that `forwardKin([0,0,0,0,0,0]) = [0,0,0,0,0,0]`

then the figures were really deviations (w.r.t 0) when 0 means 0 deviation

added an `isAroundZero` flag to choose if plot figures around 0

# 20181103
## added *plotSepFigs.m* to plot in separate plots instead of subplots
this was not really important but the plots were bigger

also added a utility/helper function to save all the plots *saveMultiFigs.m*

learned many useful things about figures

## added TCP pose sensing in all the *1D* models
it was "measured" and saved if needed

this helped me determine error in workspace corresponding to an error in jointspace

# 20181022
## modified *indeJointControl.m* response time
I used to use 0.1 s as rise time, changed to 0.05 s for faster response

I want to demonstrate that although IJC is faster than LQRwI, the performance is not better

## how to show that the 2 controller are comparable
the idea was to show they have similar performance at one task (random initial pose)

but different behaviors in other tasks (disturbance rejection)

# 20181021
a lot has happened since last unfinished update

I suffered a lot from assembling a manuscript

the research focus shifted a little bit from fast control to new control

## added *exeScript.m* for running everything in one script
this was also good for remembering what each script does and the sequence to run them

currently this was quite simple and straight forward

this was renamed as *exeDisturbScript.m* for this was for running disturbance simulations

saved it as *exeInitPosScript.m* for running simulations with a initial pose away from the origin

## added a bunch of files for 1D disturbance simulation
added *disturb1DInit.m* to init 1D disturbance and could enable noise

added *controlDemoLMwI1D.slx* and *controlDemoIJ1D.slx* for simulation

## modified *plotControlDesign.m* to plot data better
added script for setting scale in y to be the same across all subplots

the 6 plots were 2x3 now instead of 3x2 (~**TODO:** need to change position maybe later~)

## added *simLei.m* to do the simulation Lei suggested
this might not be useful later but it's good exercise nonetheless

## added *controlDemoDR.slx* for direct of dynamic rendering
I don't remember the naming convention but it's not useful anyway

## added *simTAUJADnew.slx* for testing new motions
this should be based on *simTAUJAD.slx*

I don't remember but I was trying to move it around with new motions

maybe this was an attempt to put "sensors" on TCP

## just to clear things up...
*controlDesign.slx* was the one trying to design a controller based on linearized model

this is essentially the father file of all the LQR related controller design files

*controlCompare.slx* was the one implementing IJC, father of all IJC related files

these 2 used ref models, so there was no animation

and b/c the ref was *simTAUJTDlinmod12.slx*, these are for simulation w/o disturbance on TCP

as mentioned before "Demo" files were made to have animation and later became the main version for development

# 20180907
## developed LQR controller with integrator
this part of the work was based on [this page](http://ctms.engin.umich.edu/CTMS/index.php?example=MotorPosition&section=ControlStateSpace)

as said in 0904 log, LMC was soft and this soft was generally considered as steady-state error

LQR was in its nature lack of an Integrator module, where it cannot take care steady-state error

we want the controller to reject disturbance and keep TCP at origin

## updated *linScriptJS.m* for LQR w/ I
after playing around with values, the disturbance rejection worked fine

saved a part of *controlDesign.slx* as *lqrDesignwI.slx*, played with it, and failed

added *lqrIDesignwNewPlant.slx* to implemented the idea mentioned in this link above

after testing, the controller was put into *controlDemoLMwI.slx* as the new simulation file

*controlDemoLMwI.slx* should be based on *controlDemoLM.slx* so be aware that their variables could be overloaded (bug or feature)

# 20180904
## major shift in development and test with step instead of a pulse
developed a demo on "disturbance" (sudden force/torque pulse on TCP)

this was achieved by adding bushing joint in the model and actuated by F/T

made copies of previous models and saved as a new name with a *wTT* suffix

added 6 pulses to be the input and a *disturbInit.m* to assign values to them independently

based on Lei's suggestion, extended "pulse" into a "step/square wave" by extending duty cycle

discovered that, for some reason, the models using reference model "breaks" towards the end of the simulation

major shift was stopping the development of the reference model ones and shifting to *slx* files with "Demo" in their names

same init params didn't break in "Demo" files (these files were initially for animation purpose)

the folder was growing to 70+ files and 30MB+

development would continue on *controlDemoLM.slx* and *controlDemoIJ.slx*

## performance with "constant disturbance"
"constant disturbance" was referring to the extended (to 1 s duration) disturbance pulses

current amplitudes were set to 5 N for force and 0.5 Nm for torque

IJC could resist the "press" and brought the system back to the position (origin currently) but there were 2 peaks in control input corresponding to the "step-up" and "step-down" of the disturbance

LMC seemed to be too soft as in it would be pushed away from the position by the disturbance and stayed somewhere that it found its balance again

it should be noted that currently for LQR design [Q]:R == [1:1000]:1

# 20180823
## major improvement in IJC after using new moment of inertia
after the switching, the controllers seemed to be working

but there was a requirement in rise time: if too long, TAU would not stable at one point

tested cases:
- 0.5 s, stable but with small "noise"
- 0.6 s, small wingles before "settle"
- 0.7 s, constant vibration with small "noise" (higher freq vibration)

currently, rise time was set to 0.1 s for quick and stable response

it should be noted the "trajectories" were not as smooth as LM cases

if use pole placement, similar performance could be achieved

if use lqr design, similar performance in position could be achieved with Q = [1, 1000]  (only the ratio 1:1000 mattered) 

all model could be clipped at simulation time = 1 s for details

# 20180822
(20190725) this was actually how I "linearized" IJC to obtain control plant
## started to "measure/estimate" moment of inertia w.r.t each joint
the idea was to, first fix every joint and turn the mechanism into a structure, and then spin the structure at a constant acceleration

by measuring the joint torque, estimate moment of inertia

to this end, added *structTAU.slx* to develop locked-down TAU and added *actuationTAU.slx* to actuate different joints separately

ran into "S->PS" module problem again

TIP: NEVER use filtered option; SOLUTION: provide the derivatives needed manually

more tests were needed to ensure everything was correctly implemented and measured

NOTE: about the measurement from a single revolute joint

measured total torque (3-by-1 vector) and only took the tt_z b/c that's the "actuation torque", other torques were "constraint torques"

the other sensing torque (t instead of tt) was supposed to be the combination of all torques, which was not useful in this case

## the test were carried out and results were noted
the input of this case (*actuationTAU.slx*) was capable of spin one or more joints with constant angular acc

originally, constant speed was also intended but later found no use and then stopped

manually dis/connecting the structure and the actuation was needed in the *slx*

added *IJCmomentOfInertia.m* as a dedicated script for this test

chose actuated joint and acceleration in the script

usage of the script with the *slx*: choose one joint, acc in the script; connect the corresponding ports; run script

it's verified that at different acceleration, torque over acc was a constant, which in this case was considered "equivalent moment of inertia"

the numbers were far away from what previously calculated in *indeJointControl.m*, which could potentially improve the performance of the controllers

further tests with IJC from the new params would be the next step

# 20180820
## modified *indeJointControlModel.slx* to test some theories to overcome previous problems

previously, unwanted performance (mostly large overshoot) and unpredictable vibrations were discovered with the independent joint controller(IJC)

took the assumption that the controller was only controlling the motor but not TAU, every time it should go from 0 to some wanted position

so it should work like each time a new "0" was set and the underlying assumption was that going from 0 to 1 and 1 to 2 should be the same

this solved the problem of "the controller was not behaving as designed"

## added *controlCompare.slx* to apply the IJCs to TAU
the idea was to use the independently designed IJCs to control TAU

but it's not working

taking a look back, some bold assumptions might be the problem and there was also an ultimate question

the IJCs were built based on the plant that only had acc term but no pos term

this indicated that the "load", whether internal or external, was symmetric w.r.t the motor shaft, meaning it's regardless of the position

this was far from the truth and could lead to unstable result

compared with the LM controller, the "info" about init position was mostly lost due to the fact that no matter where it started, as long as the "displacements" were the same, the controller couldn't tell the difference and the output of the controller was the same

init TAU at the origin was also tested and the controller couldn't hold the system at the origin either

faster controller with faster ob (for robust to modeling error) was also tested but it didn't work

one thing didn't try was disturbance analysis, where the controller was robust to disturbance (everything that's not considered in the plant was considered disturbance)

as mentioned, since nothing worked, the ultimate question surfaced: what if this would never work

# 20180815
## added *indeJointControl.m* and *indeJointControlModel.slx* for independent joint control for comparison
there were 2 ways to make a comparison with my newly designed controller

one was against a full theoretical dynamic model (on hold, again), the other was against a independent joint control structure (WIP)

as the name suggested, this was control single joint w/o considering coupling effects among joints

for the moment, the plants for all joints had the form of 1/s^2

PID with LP (anti-windup) was implemented as the controller

this was copied from previous workshop 1 with continuous & discrete implementations

the result was a bit unpredictable

~currently, one of the problems was to design both controllers under the same requirements~

agreed on creating controllers with similar response time, then compare other aspects of the controller like control effort and stuff (20180820)

another problem seemed to be a implementation problem: there was a unwanted vibration at the initial position

# 20180729
## not sure if "piece-wise linear" ever made sense here but this was about SM and LM development

## added to take care of the plot
previously, the data logging plot was not satisfying and flexible enough

added many "to workspace" blocks to all the logging signals in *controlDesign.slx* for later plots

added *plotControlDesign.m* to plot position, velocity, and input

it seemed that though OP was at the origin, TCP could still go back from a far point on the edge of the workspace

## some unlogged stuff 
added *controlDesign.slx* for further controller design

added *controlDemo.slx* for animation of the controller performance

this could not be achieved by *controlDesign.slx* mainly b/c a ref model was used

otherwise, there was no big deal with this demo file

added a RefPt in *controlDemo.slx* to verify the displacement in TCP

in *linScriptJS.m*, commented out the sign flip for init_joints_p_sim b/c there was already a sign flip hard coded in joint 4 in *simTAUJTDlinmod12.slx* already 

maybe it's better to left *simTAUJTDlinmod12.slx* unchanged

# 20180604
## info on motor selection in TAU
as stated in the MF 2004 report and other materials, the motor used was RE 25 graphite brushes, 20W, 339152

with nominal torque 28.8 mNm and stall torque 304 mNm, more specs could be found in catalog

gear ratio: joint 1 & 3: 10; joint 2 & 4: 3.5;

# 20180529
## LQR tryouts
"intuitive values" for QR didn't give fast response and tuning from there seemed hard

dropped back to equal weights and settling time was around 10 s (still slow)

part of this was following the routine on [this page](http://ctms.engin.umich.edu/CTMS/index.php?example=InvertedPendulum&section=ControlStateSpace)

in Q, set position to 100 and velocity to 1. the settling time was below 1 s

set velocity to 0 would lead to some small oscillation

the input amplitude were also OK in this case

changed 100 to 10 would get a settling time of 2 s

## pole placement tryouts
with `place`, the same pole couldn't be picked for more than rank(B) times

`acker` can solve this but `acker` is not applicable when order > 10 (added 20190527)

# 20180523
## developed "sensor" for TCP pose
added a bushing joint in *simTAUJADnew.slx* and tested with modified *testMotions.m*

the reason behind this attempt was the hope of controllable system with a change of state/output (from JS to WS)

accordingly, position and orientation of the TCP and its derivatives were chosen as state/output

this gave rise to a series of files

*simTAUJTDlinmod12.slx* and *simTAUJTDlinmod12NoG.slx* were JS as state and output

renamed *linScript.m* as *linScriptJS.m* to be the script for these models

saved *simTAUJTDlinmod12.slx* as *simTAUJTDaddOs.slx* and copied the new TCP sensor in to have 24 output (JS and WS)

this model was used to develop the LM with JS as states and WS as output

saved *linScriptJS.m* as *linScriptWSJS.m* b/c of the further transformation

saved *simTAUJTDaddOs.slx* as *simTAUJTDchangeOs.slx* and deleted the output related to JS (only 12 output for WS)

the state/output for this model were WS so C matrix is I, the same as pure JS models

saved *linScriptJS.m* as *linScriptWS.m* for this specific model

after discussion, we would still push on *simTAUJTDlinmod12.slx* and *linScriptJS.m* for now

## current problems with the SM/LM (solved)
the result from ctrb() was still uncontrollable

there was still a possibility that there was a bug or a mistake

however, after reading the reference page of ctrb(), ctrbf() was recommended for some error-sensitive cases

from the result from ctrbf(), the LM seemed to be controllable, which is consistent with controllable LM reported in previous control design

this was true for all gravity-enabled models

~if it was controllable, then the remain problem was the controller performance on SM~

this was not a problem anymore, mystery solved by Lei, which was a mistake on my behalf

the current conclusion was although ctrb() returned uncontrollable, it is, according to ctrbf()

previously, SM wasn't doing so well b/c input was missing initial condition values

we could work on gravity-enabled case after this

## gravity-enabled LQR design with some intuition
literature said Q was related to max state value and R max input

range of joint_angles was found and range of velocity was estimated by range divided by time

the range of joint_torques could be related to the torques to keep it at ep/op

it could also be applied by a safety margin, in this case, 50% extra

these could be combined and a constant value could be found

it should also take the max torque from motor into account

but since there were transmissions, the assumption was that saturation was not a problem (not really)

there was no direct relation between QR and poles, hence the response time

currently it took less than 10 s for LQR controller to converge

## about definition difference with joint 4
the definitions in IK related stuff and AM were the same, joint 4 axis point towards main column

the definitions in SM related stuff were the same, pointing out as joint 2

it's too late now but maybe this should be avoided in the first place

secondly, every time after IK routine, a sign flip should be done and saved separately with a noticeable name

thirdly, a flip could be done in IK related routine

but now, there were just more blocks in simulink to correct them

# 20180522
## updated *simTAUJTDlinmod12NoG.slx* for initial conditions manipulation
started controller design and adopted the routine in Lei's lecture notes on inverted_pendulum 

the ability to set initial conditions of all states (velocity and position) was needed

treated "init_joints" as a params for op specifically, related to establishing LM

updated the variable names in all the active joints in *simTAUJTDlinmod12NoG.slx* and added velocity initialization

renamed *simTAUJTDlinmod12.slx* as *simTAUJTDlinmod12old.slx* just in case there was some changes from *simTAUJTDlinmod12.slx* to *simTAUJTDlinmod12NoG.slx*

saved *simTAUJTDlinmod12NoG.slx* as *simTAUJTDlinmod12.slx* and activated gravity to update *simTAUJTDlinmod12.slx*

## preliminary control design using place() and lqr()
added *controlDesign.slx* to test out controllers (gains) generated by these 2 command

this was the first file in this direction without a suffix for  method or model specifications

this was currently for both w/ or w/o gravity and for both pp or lqr design gain

w/o gravity, these 2 commands worked on a simple test case

init_TCP_pose_sim = [0; 1; 0; 0; 0; 0] and init_joints_v_lin = [0; 0; 0; 0; 0; 0.001]

pp had faster response (within 1 s) than lqr (within 5 s) with higher gains so there was no conclusion about which method was better

w/ gravity, though reported uncontrollable, the 2 commands still executed and LM seemed to converge to origin

however, SM was not doing well. 

half of joint velocity went to (near) 0 while the other half seemed to have a steady-state error (SSE)

according to this, half of joint angles had SSE and the other half didn't converge

# 20180521
who are we kidding, she is just gorgeous

this is more like a weekly update now

define: AM, SM, LM to be ADAMS, Simscape, Linearized model

## more on what's next
it was confirmed that for some unknown reasons LM was uncontrollable with gravity activated

3 things to try after this point:
- do the whole thing w/o gravity to see what happens

- separate and get rid of the uncontrollable part of the system

- change output from joint space to TCP pose

- figure out why it's uncontrollable

(20180525) although this was already solved at this point but I happen to stumble upon a way to [identify uncontrollable modes](https://se.mathworks.com/help/control/ref/tzero.html#bs9mcm_-1)

## saved *simTAUJAD.slx* as *simTAUJADnew.slx* for motion testing
checked with *simTAUcheckOTorq.slx*, SM was at an odd pose though input angles should be correct

this could related to the multiple solution of the simulation and singularities

no good solutions were found except adding hard stop to joints, which was tedious to implement and probably not what we needed

attempted to try this with continuous motion (*simTAUJAD.slx*) instead of sudden initialization (*simTAUcheckOTorq.slx*) and it worked (there was no sudden flip or crossing singularity)

since the model changed a lot since *simTAUJAD.slx*, an updated version was created

the test script was *testMotions.m* (added), similar to the routine in simTAUInit.m/load dataGen - use with Joint Angle Driven

tested with the updated model and it worked fine in motion (z varied from 0 to 10)

but the problem would show when initialized from 6 and moved to 10

# 20180514
## updated verification framework for input signal type and op
currently signal types were: step, pulse, sine wave, and square wave

according to results, most tests were done with step

data-logging was grouped by mux for better naming

there was still a "zero alignment problem" across models

~**TODO:** patch this in later implementations b/c it's not that important~

## tried different things to understand and verify the models
though tried different things with input, the dominant factor seemed to be "average amplitude" if under same amount of simulation time

at first, all tests were done with a simulation time of 1 s and a delay of 0.2 s in the beginning

then, a few test on very large amplitude was done in 1 s with a delay of 0.1 s

after that, different ops were tested for delay of 0.1 and simulation time of 1 s and later 0.6 s for shorten the overall time

~too many plots were generated but different behaviors were observed between ADAMS and Simscape~

the results for ADAMS model might not be applicable b/c ADAMS model didn't have the capability to initialize at different ops

ADAMS model could be starting at the origin and that might actually be the reason behind "zero alignment" problem

# 20180507
## moved verification framework to this repo
saved *stepTestNoG.slx* as *signalTestNoG.slx* and moved it here

renamed *simTAUJTDlinmod12.slx* as *simTAUJTDlinmod12NoG.slx* and moved it here

the *simTAUJTDlinmod12.slx* currently was with gravity

copied ADAMS related stuff straight from ADAMS folder

there was a flag to choose from step, pulse, or sine wave

compensated the output from the lin model

## tested step and pulse input
the results from step and pulse were quite the same

it's still a bit strange when joint 5 was active (could also be a sign flip)

## tried to determine the uncontrollable state/mode but failed
the idea was to transform to modal form but then the states had already changed

besides, the system was sometimes controllable

with gravity, with correct torque input, at origin, uncontrollable

with gravity, with zero torque input, at origin, controllable

zero gravity, with correct (=zero) input, at origin, controllable

I could have done something wrong and, besides, it's not that important (controllability)

# 20180501
## tried different ops for ctrb() amd obsv()
first of all, 10 instead of 12 needed more reading to understand

tried different op with a different TCP_pose and corresponding input (joint torque) but nothing changed

tried to understand this from mechanism point of view (DOF, constraints) but failed

# 20180429
most of the info would be in this log for today

the work of today was to bring Simscape model closer to ADAMS model

specific work included going back to ADAMS, CAD model to retrieve some numbers and making some simple reasonable simplification

## updated *simTAUcheckOTorq.slx*, *simTAUJTDConstIn.slx*, and *simTAUJTDlinmod12.slx*
these were the most up-to-date model for the time being

added BD bar (L3), hook joint connector, materials, L1 L2 real length

added the handle on the platform and moved the platform away a little from the TCP (using angle_beta)

utilized hookOffset (current understanding was that it's just a way to set a better 0 for the joint b/c of range limit)

however, it was visible that, at B and D, the alignment was not perfect

the same should be true for implementing the sphere joint (hook + rot) close to the platform

to visualize this, these offsets were there so that everything aligned beautifully when TCP is at origin

## found more discrepancy between models (some facts)
although there were materials assigned in CAD model, they were not the same (density) in ADAMS (ADAMS as standard, always)

L1 bars in CAD were hollow tubes but they were solid cylinders in ADAMS

the L1 bar in chain 3 was longer than the other two, see ADAMS for details

only the L1 bar in chain 1 was defined as carbon fiber in ADAMS (all were steel in Simscape)

all bars in Simscape were defined with the center of mass (CM) at the mid point of the 2 endpoints (this might not be true in ADAMS)

based on all these, init_torques_adm = [-359.3; 217.8; 359.5; 2.40; -0.22; -952.1] and init_torques_sim = [0.3630; -0.2233; -0.3630; 0.0063; 0; 0.9414]

# 20180426
for some reason, the z axis of joint 4 in the ADAMS model is not point out.

this introduced a sign-flip between ADAMS model and Simscape model

so the shared data between the two models needs extra care while the data used in only one model doesn't need that

## deleted the -1 gain in joint 4 in *simTAUJTDlinmod12.slx*
b/c the two model has different mass params, joint torque calculations only make sense in its own model

so -1 gain in all JTD related *slx* files should be deleted

# 20180420
the linearization of the system seemed finally working

the next step was to verify the linearization

real robot -> ADAMS model -> Simscape model -> linearization of the Simscape model

to verify, all models should be as close as possible on params and stuff

## updated *simTAUInit.m* to have the correct params
opened ADAMS model and updated the geometric params to be the same

noted that most parts in ADAMS were actually defined as "steel"

noted that BD was not modeled in Simscape model and it should be "aluminum"

after this update, the torque at the origin looked closer to that observed in ADAMS

# 20180418
## added *linScript.m* to define options and operating point for linearize()
switched from linmod() to linearize() b/c it seemed better

saved *simTAUJTDlinmod.slx* as *simTAUJTDlinmod12.slx* and included joint velocity as well as joint angles as system output, making #output 12

linearize() gave me a system with 12 states and with 12 outputs, C matrix became 12x12

based on Lei's help, if C is invertible and if D is all-zero, the problem could be reformulate so that the state is the output, making the new C matrix an Identity Matrix

this was tested and possibly doable and this would make the states the same as what we desired

the code for this reformulation was implemented in the *linScript.m* script

the current problem was to find the right operating point (op)

findop() always returned a msg "Could not find a solution that satisfies all constraints.  Relax the constraints to find a feasible solution."

that being said, the supposed op after this was "quite close to" what was designed

a way to verify or make sense of the resulted system was needed

no good documentation was found for this specific feature

## saved *simTAU.slx* as *simTAUcheckOTorq.slx* to see the torque needed to hold at the origin
sensing joint torque was added in this new file

assumed the 3rd value from the total torque was what needed

the idea was to JAD, sensing JT, then JTD to see if the TCP will stay at the origin

saved *simTAUJTDlinmod12.slx* as *simTAUJTDConstIn.slx* for this purpose but it didn't work

~the system wouldn't stay for no good reason (bad reason: numerical errors and stuff)~

solved by frustratingly flipping the sign (this was a good reason)

the system would stay for almost 1 second (this could be due to numerical stuff)

control problem: after a small disturbance, how to come back to this state or stay at another state.

# 20180417
## renamed the file *simTAUJointDriven.slx* as *simTAUJAD.slx*
4 signals of the system: joint angles, joint torques, TCP pose, TCP F/T

one should be the input and the rest could be determined

so there would be 4 files with *simTAU* as a prefix and the last 3 letters specified the input (the system was driven by this signal)

all the other files should be named accordingly (JAD: joint angle driven)

~**TODO:** the sensor output had not been defined in this file yet~

## saved *simTAUJAD.slx* as *simTAUJTD.slx* to prioritize the model development for linmod()
the model wanted for linmod() was JTD: joint torque driven

flipped the I/Os for all joints with input torque and joint velocity sensing

save this as *simTAUJTDlinmod.slx* for further development

## used linmod() over *simTAUJTDlinmod.slx*
the result was not satisfactory

it automatically chose many states including the ones w.r.t the passive angles

it also treated initial input strangely

found another command linearize() which seemed to be more advanced and flexible

but still states couldn't be specified

# 20180416
## confirmed the same convention was used for our code and the bush joint
see [the other repo](https://github.com/easyt0re/Kinematics_Rewrite) for more details

but I had trouble finding a simple visual demo to support this

stopped the development of *inputSourceTCP.slx*

# 20180411
## patched the rotation before the passive hook joint
previous Simscape models didn't take into account the (90 + hookOffset) rotation at B1 and B2

this was implemented so that, without the TCP platform, the elbows for limb 1 and 2 were still bent

hookOffset was currently set to zero

**TODO:** figure out on what level this hookOffset will take effect

## fixed the problem with the discrete input in *simTAUJointDriven.slx*
*simTAUJointDriven.slx* was added to drive the system from joint angles

it was working after a few tries with variableStep Solver

the S->PS option should be "filter input, derivatives calculated" and "second-order filtering"

this was working currently so didn't dig deeper

since this was intended to simulate physical system, continuous stuff should be preferred

used to guess the type of input was wrong, so tried lookup-table input. should be the same, comment out instead of deleting it.

added *inputSourceJoint.slx* to try out different kinds of input, could be deleted later

## added *simTAUTCPDriven.slx* to develop a ["indirect inverse dynamic"](https://blogs.mathworks.com/simulink/2013/11/13/motion-actuation-in-simmechanics-r2013b/) model
this was needed in order to get joint torque profile to drive the system in a forward manner

a bush joint was used to define orientation of the TCP but the convention of this was unclear

added *inputSourceTCP.slx* to study the convention

# 20180404
## paused the exploration of linmod()
current assumption was that linmod() + ADAMS was not working

a model directly built in Simulink was needed

## started Simscape modeling of the system
added *simTAU.slx* with the pairing initialization script *simTAUInit.m*

### TODO LIST:
- ~move it a little~

- update the correct params of the robot and verify against ADAMS

- ~check the elbow part of the robot and recover most of the features~

- setup the correct I/Os for different purposes afterwards

- (optional) inversely driven model with limits

# 20180330
## updated the ADAMS model to input: joint torque output: joint angle
linmod('model') would run into error

tried to provide equilibrium point but the error stayed the same

one reason could be that the accuracy of the equilibrium point was not enough so "it could not initialize"

another question was how the linmod() function could tell the "state vector" of the system

my system should be x = [joint_angles; joint_angles_dot], u = joint_torques, y = joint_angles

## tested nonlinear hydraulic system with linmod()
it worked pretty well and it generated states on its own

but this counted for nothing b/c the model was built in Simulink where there were explicit integrator block to help it determine states

still, how it determined states and how it matched states with initial conditions were pretty magical

it could add extra states according to its own algorithm

## used linmod() on the ADAMS model
the previous problem seemed to be that there were multiple "control" modules active

disabled all the rest and export control plant again

linmod() could run but it gave back nothing useful

the "state" of the system seemed to be 1 and couldn't be changed

this might indicate that linmod() cannot be used over ADAMS model

# 20180325
as stated in the other repo, this work would have its own repo and this is it

new to the field, much knowledge needed

ran MATLAB linmod() on the ADAMS control plant model and it actually worked

but the result was not what I was looking for

my current understanding of the command was to generate a state-space linear model

so, the state of the system should be clear

but I think there is a difference between MATLAB and ADAMS

the input of a MATLAB model is a sequence of numbers (or vectors) which doesn't have time stamps, meaning the time between two entries are not defined. the iput is not a "trajectory". there is no time information.

output = matlab_model(x, x_dot)

the input of an ADAMS model is a trajectory, a sequence of time and number pairs. the derivatives are calculated inside the model.

output = admas_model(x(t)); x_dot and x_dot_dot are calculated locally inside the function

this difference in the function interface is the current supposed problem I have

another thing could be that I choose the wrong I/Os, AGAIN!!!
