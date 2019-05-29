# piecewiseLinearModel
This is a log for the development of the piece-wise linear model of our system

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

added frame viz representations at origin and TCP. the one at origin (not moving) was darker

test log: `linop = [t_x, t_y, t_z, r_z, r_y, r_x];` 

- to have non-zeros in all joints, we changed `r_z`. with everything else sets to 0, `r_z` could range from - 0.1 to + 0.3 with step of 0.1. - 0.2 and + 0.4 led to strange IK solutions. + 0.5 broke the assembly. 0.3 (rad) is around 17 degree

- after a closer look, it's not the problem of IK but of simulation. if TAU was initiated at, say - 0.3, directly, TAU went into "singularity". however, if it initiated somewhere else and moved to - 0.3, everything was fine, proving IK is correct. 
TODO: more reliable initiate process or some limits on passive (spherical) joints. initiation is used in static torque computation and start the simulation

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

- [ ] quantization in encoders and check other aspects of discrete design

## change the "structure" of the code
- [ ] implement things as functions instead of scripts with no input args

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
- [ ] quantization of sensors in Simulink model

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

**TODO:** how to calculate poles for MIMO or what does my calculation mean?

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
- 1:100:1, overshoot was even smaller but the recovery from the overshoot was too slow, the rise time was quite fast, after 0.5 s it's almost at the ref with a bit overshoot

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

the 6 plots were 2x3 now instead of 3x2 (TODO: need to change position maybe later)

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

TODO: patch this in later implementations b/c it's not that important

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
for some reasons, the z axis of joint 4 in the ADAMS model is not point out.

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

**TODO:** the sensor output had not been defined in this file yet

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

TODO: figure out on what level this hookOffset will take effect

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
