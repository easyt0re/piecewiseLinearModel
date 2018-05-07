# piecewiseLinearModel
This is a log for the development of the piece-wise linear model of our system

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
