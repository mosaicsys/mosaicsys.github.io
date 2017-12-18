# Interlocks Post V1

## Overview

The term Interlock is generally used in a mechanical sense to describe systems and mechanisms where the functions of closely related parts must be manipulated in such a way as to preserve common constraints.  In its most basic sense it means that the conceptual "fingers" from one part are "interlocked" - interdigitated in some way - with the fingers of the other.

In an Equipment Control Software (ECS) sense this term carries additional, and much more specific, meaning.  Here the term relates to the set of criteria that must be designed and enforced by the software to address electro-mechanical constraints of the equipment so as to attempt to avoid causing damage to the tool and/or to the customer material that it is being used with.  Please note the explicit exclusion of protection of damage to people as the types of software that are being contemplated and discussed here are not generally certified for use in personal safety related logic systems (S2 compliance, etc.)

Within the confines of software enforced rules to attempt to prevent causing equipment and material damage that could reasonably be prevented by the software, we find two basic types of such interlocks: on request, and continuously observed.  

On request interlocks are generally designed to be used at some point of request, either from the equipment's current user (or other external decision authority), or from other internal sequencing logic in the software, and these interlocks generally serve to validate that the ECS believes that the request is currently acceptable and then either initiate it or reject it if it is not currently acceptable.  

Continuously observed interlocks serve a somewhat different purpose in that they repeatedly observe the state of the equipment and attempt to recognize state changes, typically triggered by conditions that are outside of the ECS's control, in which damage could be caused to the equipment if it is left in this new state for a sufficient period of time, and for which there is at least one trigger able action that the ECS can take that might help prevent cascading or compounding damage to the equipment or to customer material.  These continuous interlocks are generally used to recognize conditions where the operation of one component of the equipment is conditional on continued correct operation of another and when the second is found to have failed to maintain the condition it is responsible for, the interlock is used to change the former's state (usually by turning it off) so as to attempt to prevent cascaded damage.  Continuously observed interlocks generally include logic to support a time and continuity hold-off constraint so that they do not trigger in response to a certain level of noise on measurement and other electrical inputs.

Both on request and continuous interlocks share the concept of a tree of conditional state evaluations that are performed against the state of the equipment as currently observed by the ECS.  For on request interlocks this condition tree evaluation is used to trigger acceptance or rejection of the request.  For continuous interlocks this condition tree is repeatedly evaluated and when a result indicates, a reaction is triggered to attempt to mitigate and prevent cascaded damage, using the most reasonable means available.

The focus of the remainder of this document will be on ECS implementation of interlocks (mainly of on request interlocks) as they apply to semiconductor capital equipment, and to the vacuum systems that are commonly used in such equipment.

At this point it is likely very apparent that such ECS interlock implementation approaches will have many "if" statements.  The nature of this type of logic is that its behavior is highly conditional on the exact state of the equipment at the point that a specific trigger condition is evaluated.

## Equipment specific details

For any given piece of equipment, the set of reasonably useful interlocks for it will have significant overlap with other types of equipment that share use of certain types of components and common component configurations.  However the details of such interlocks, as implemented, almost always include equipment specific characteristics.  The set of interlocks for a specific type of equipment, in total, will necessarily be very specific to the details of that equipment's electrical and mechanical design. 

## Interlock implementation goals

In this discussion there are a number of high level goals that we are attempting to accomplish:

* Single point of evaluation for any given condition: In any given condition tree, there shall be exactly one source code expression that is used to perform each boolean condition from which the tree is built.  There may be many object instances that make use of this line in different contexts but they must share use of a single compilation unit that contains the compiled result from that line of code.
* Similarly the logic used to combine individual expressions into a logic tree shall be done to minimize the redundancy in the structure of the tree as much as possible.  
* Self descriptive failure codes:  Generally the results from any failed condition evaluation shall be a string description of the failed condition which, where appropriate and feasible, shall be able to include text representations of the underlying values that failed to meet the desired condition.
* Support hierarchical evaluation so that different code paths through the ECS may make use of both common and path specific condition evaluations so as to support both internal protection for production use patterns and to support more complete user service action protection using a hierarchical combination of the production use case condition tree and the service action specific service tree.  
* Support the ability to report when more than one "deny reason" is active for a given interlock condition tree so that the user interface can use this logic to report a more complete set of current conditions that are preventing a requested action from being performed.
* Provide per-request support to allow a suitably authorized user to block applicability of a given deny reason or set thereof when processing a specific request.  Generally this behavior shall be managed in the GUI and shall be propagated to the business logic in such a way that the business logic can remove explicitly selected deny reasons for a current set of them using information provided with the request so that the business logic can confirm that all currently active interlock reasons have been suppressed explicitly when allowing the user to bypass these reasons for this request.  Generally we expect to use StartsWith, Contains or Regex based rules to allow the user interface to select which deny reasons it is explicitly requesting the suppression of.

Please note that for continuous interlock purposes the code may support per call optional generation of state dependent condition strings.  This will allow the interlock logic to make use of static (single instance) constant strings for the repeated condition evaluation logic up and until the point at which the trigger logic and hold-off are about to be met, at which point the final condition confirmation shall allow the addition of state dependent parts (read back and configured threshold values).  In this way the logic can generally honor the single point of implementation concept and the consolidated condition tree logic flow concept without generating a large volume of dynamically generated string content and thus adding significantly to the business logic application's memory garbage generation load.

## User Interface Goals

Interlocks may be rendered in a ECS's GUI in a number of ways.  One historic way reminds me of the HitchHikers Guide to the Galaxy in which Ford presses a button on a ship and a small sign lights up that says "Please do not press this button again".  It has historically been common place to accept GUI control click (or activation) and then perform interlock based validation of the request and finally produce a failure message, in the form of a message box, if the request validation failed.

In general we find that these re-active representations of interlocks are less than ideal, although there are some cases where this approach remains the only viable one.

Instead we prefer to see the GUI animate all (or most) user selectable controls so that they are disabled when they are not selectable by the user for some reason.  This is then combined with the ability to produce a Tool Tip for the disabled control so that the user simply hovers the mouse over the control to see why it has been disabled.  

Control disable logic generally represents a larger tree of ECS enforced interlock tests including information about the permissions available to the current user and the state of the user's ability to actively control the equipment through the GUI (view only vs control modes).  Given that the internal production oriented parts of the ECS are not generally constrained by, let alone aware of, the current logged in user(s) or the permissions that have been granted to that user or users, it seems reasonable to infer that some aspects of any given selectable control need to support local applications of user specific and/or GUI configuration specific constraints, in addition to the ones that are produced directly by the underlying business logic. 

### Use On Touch Screen Based Hardware

This approach does, however, presume that screen is mouse based and not touch based.  Depending on available touch hardware there are a reasonable set of alternative techniques to support similar concepts on touch only user interfaces.  These include means for the user to dynamically modify the meaning of normal "click" type touches and/or the use of custom gestures to cause a disabled control to expose any related tooltip.

For example a "shift" type key or screen control could be used to temporarily reconfigure the screen so that clicks on previously disabled controls now serve to explicitly display the "Tool Tip" information that is available for that control, rather than to attempt to activate the control in the normal manner.

## On Request Interlocks

Based on the above discussion, we end up with the following uses for on request interlocks:

* At a low level evaluate, and conditionally, block requested actions (from any source) so as to attempt to prevent causing equipment and material damage when the ECS could reasonably be able to prevent it by blocking the requested action.
* Disable various user screen controls (usually buttons) when such a control is known to be unusable based on the most recently observed equipment state and provide Tool Tip style indications of at least one reason why the control cannot currently be selected/used.

Example:  You cannot turn enable the high voltage (HV) supply unless the chamber pressure is known to be either higher than or lower than of the critical avalanche breakdown pressure (based on Paschen's curve) such as >= 100 Torr or <= .1 Torr.  In this case we would expect the HV supply enable.

## Temporary suspension of on request interlocks

In addition the GUI is generally expected to provide means so that a suitably authorized user can temporarily re-enable controls that have been disabled due to deny reasons from the business logic (but not those due to user permissions, etc.).

For example the GUI can provide such a user with a interlock behavior slider or a keyboard button that flips the screen into a mode where controls with active business logic sourced deny reasons would be visually annotated to warn the user of which controls have such active deny reasons and, when the user activates any such control, the GUI would explicitly prompt the user to confirm that they would like to request that the underlying operation be performed anyway.

Any such mechanism should require much more specific user attention (like holding down a key on the keyboard) and, when supported, the screen should clearly display this alternate, safety degraded, use mode using clear indications.  We would expect that color (orange or yellow perhaps) and warning tape banner annotations (yellow and black diagonal strips with Warning emblem(s)) might usefully serve such a purpose.

## Continuous Interlocks

Continuous interlocks generally consist of the following features/characteristics:

* One or more entity that is tasked with the periodic evaluation of each interlock's condition tree and trigger hold-off 
* The condition tree logic itself and the specific trigger hold-off value.
* The triggerable (re)action that is to be used when the condition tree is triggered and the hold-off delay is met. 
* Means to configurably and/or programmatically (timer?) disable evaluation and/or triggering for one or more interlock (usually this is done as all interlocks of a set based on general type and/or sub-system).  When such interlocks have been disabled there shall be a clear indication on the relevant screens (warning annunciator?)
* Support for some form of annunciator to indicate when a given interlock has triggered.  This may be informational, warning or alarm based on the condition and the frequency that it is expected to be triggered at.

As with the on request interlock example, an example here would be that the same HV supply would be abortively disabled if the chamber pressure was measured to be within the critical avalanche breakdown pressure window (.1 Torr to 100 Torr) for at least .1 seconds.  This would trigger the HV supply driver to disable the supply.  It would likely post an alarm annunciator as this is not an expected behavior.  Based on frequency of this failure mode and on the supplies ability to handle any resulting arc condition, this interlock's hold-off period and/or the severity of its annunciator might be adjusted.

## Implementation Approaches

For this portion of the discussion we will assume that the relevant logic will be coded and implemented to generally follow patterns supported by MosaicLibCS including the use of the SAP (Simple Active Part) pattern and annotated Value Sets, IVI's and IVA's.

Our primary design goal is the support for single point of implementation for conditions and condition tree's.  As such the support for screen button disable logic might become challenging.

### Version 1

The first, and most obvious, general approach here for on request interlocks is to have the business logic parts continuously generate and publish individual disable reason sets for each unique set of screen controls, or at least a reasonably efficient choice of deny reason sub-tree sets from which the screen code can joint to form the actual per control sets.  This approach does have a major drawback which is that it saddles the business logic code with the cost of generating and publishing a relatively large set of deny reason sets, each of which is likely to include noisy equipment state information (pressures, flows, ...).  

As such this generation and publication could easily become a significant computational, memory garbage and communications load on the system, especially if the GUI is running in a separate process from the business logic.  There are obviously reasonable means that can be used to minimize this cost such as lowering the condition tree evaluation rate, blocking inclusion of dynamic state and/or portions of the condition trees when a part is being used in production (most related deny reason sets can simply indicate that the part is not in a usage state that supports service actions).  

This general approach easily supports the goal of single point of condition and condition tree implementation.

### Version 2

Another, somewhat less obvious, approach for on request interlocks is to render the majority of the business logic centric condition and condition tree evaluation logic in separate helper classes that support instantiation and use of the condition and condition tree logic in both the business logic parts and in the GUI code directly.  

In this way the business logic instantiates one or more interlock condition helper objects per part and uses them as a pre-condition check on individual actions (both internally sourced and externally requested).  These checks are literally performed on request and as such do not cause the same level of redundant repeated evaluation as the prior approach would be expected to.  

Then the GUI also instantiates the same set of interlock condition helper object(s) and periodically evaluates them on a timer.  This GUI version will need to be able to generate a bindable version of the outputs from each of the related deny reason set condition trees.  In addition the GUI version of this logic will need to have access to the same set of required equipment state information.  Generally we expect that these requirements will be reasonably met by the replication of a common IVI from the business logic to the GUI and by the careful use of Value Set objects and custom Value Set Item Attributes so that the primary difference between the business logic and the GUI version of the helper instances is exactly where they get the Value Set object contents from that is used to support evaluation of their condition trees.  

We expect that the majority of the condition tree logic that needs to be common between the business logic and GUI will be based on IVA's that are observable in both the business logic parts and in the GUI with little custom interlock specific work required to accomplish this.  The notable concern with this approach is that the business logic might need to double define specific value set items that are used this way or use different bound value propagation direction logic than is used in the interlock helper class.  Likely the most efficient means to render this approach is to compose the state information that is required for a given set of interlock condition tree's from the annotated properties in a small number of value set objects.  These objects are typically defined so as to support both a set of business logic and the set of on request interlocks (and possibly also continuous interlocks) for that set of business logic.  Then these are further split into additional value sets based on the directionality of the value flow in the business logic.  The corresponding interlock helper class will be provided with fresh value set contents by the environment in which it is constructed (business logic part in one case and by the timer triggered GUI logic in the other).

## End of document