# piecewiseLinearModel
This is a log for the development of the piece-wise linear model of our system

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

with norminal torque 28.8 mNm and stall torque 304 mNm, more specs could be found in catalog

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
the same pole couldn't be picked for more than rank(B) times

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

secondly, everytime after IK routine, a sign flip should be done and saved separately with a noticable name

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

no good solutions were found except adding hardstop to joints, which was tedious to implement and probably not what we needed

attempted to try this with continuous motion (*simTAUJAD.slx*) instead of sudden initialization (*simTAUcheckOTorq.slx*) and it worked (there was no sudden flip or crossing singularity)

since the model changed a lot since *simTAUJAD.slx*, an updated version was created

the test script was *testMotions.m* (added), similar to the routine in simTAUInit.m/load dataGen - use with Joint Angle Driven

tested with the updated model and it worked fine in motion (z varied from 0 to 10)

but the problem would show when initialized from 6 and moved to 10

# 20180514
## updated verification framework for input signal type and op
currently signal types were: step, pulse, sine wave, and square wave

according to results, most tests were done with step

datalogging was grouped by mux for better naming

there was still a "zero alignment problem" across models

TODO: patch this in later implementations b/c it's not that important

## tried different things to understand and verify the models
though tried different things with input, the dominant factor seemed to be "average amplitude" if under same amout of simulation time

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

noted that BD was not modeled in Simscape model and it should be "aluminium"

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

solved by frustratedly flipping the sign (this was a good reason)

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
