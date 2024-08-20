---
title: Mechanism vs. Policy
category: Software Engineering
---

In [The Art of Unix Programming](http://www.catb.org/esr/writings/taoup/html/){:target="_blank"}, Eric Steven Raymond discusses the Unix approach of avoiding enforcing policies and preferring to define a mechanism while leaving the policy definition to the user.

This is an instigating perspective, so I would like to explore the difference between mechanism and policy a bit further.

According to the Cambridge Dictionary, a mechanism is "a way of doing something that is planned or part of a system". So, it is *how* the system accomplishes a goal or the configuration of its internal parts that allows it to reach this goal. For instance, two vehicles may have the same goal of transporting people but work on completely different mechanisms (say a combustion engine and wind.)

The policy, on the other hand, again according to the dictionary, is "a set of ideas or a plan for action followed by a business, a government, a political party, or a group of people." So it is an agreement of how to behave or set things up that should be followed. Under the risk of stretching the metaphor too far, in the vehicle example, a policy might be the steps to build a tool to give direction to the vehicle and connect it to its internal mechanism. If this internal mechanism provides a generic interface, one user might decide to build a steering wheel, and another might go with a joystick.

Such a generic interface gives power to the user, who doesn't have to be constrained to one way of doing things. However, as Eric Steven Raymond puts it, "When the user can set policy, the user must set policy." If the vehicle only accepted one kind of steering wheel, maybe all the user would have to do is buy one and plug it into the internal mechanism. With a generic interface, however, they will have to figure out many details.

Apart from less freedom, the flip side is that policy tends to change more often than mechanisms. So, a system that enforces policy will likely undergo frequent cycles of needing to be updated.