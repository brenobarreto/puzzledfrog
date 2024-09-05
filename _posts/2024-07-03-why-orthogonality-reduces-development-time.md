---
title: Why Orthogonality Reduces Development Time
category: Software Engineering
---

Back in the 1999 edition of The Pragmatic Programmer, David Thomas and Andrew Hunt said that orthogonality was a concept rarely taught directly. Instead, it was an implicit feature of other methods and techniques. This means that if you ended up building an orthogonal system, you would do so as an unconscious side effect of using a method that happens to enforce orthogonality. The opposite would be to make your system orthogonal intentionally.

Twenty-five years later, I feel that this hasn't changed much. Even among experienced software developers and architects, I rarely hear the word "orthogonal" applied during the design of new systems or in the analyses of existing ones.

I'd like to reflect on why it's useful to explicitly think in terms of orthogonality. But first, let's define the term using a simple example. Imagine you are driving your car, and the radio's volume increases when you step on the accelerator. When you release it, the volume decreases. This means these two parts of the system are coupled: one change in one part affects the other. Another way to describe the relationship between these two parts is to say they are *not* orthogonal.

Bringing this concept closer to software engineering, we might think of a layered system in which a simple change in the presentation layer requires a change in the database model. We can easily see how this presents an issue.

As The Pragmatic Programmer states, "nonorthogonal systems are inherently more complex to change and control." That's because the benefits of modularity are greatly weakened in these systems. You cannot isolate a change in one part of the code because that change will affect another part of the code. So, local fixes become impossible, and complexity rises.

To build orthogonal systems, you need to think in terms of isolated components that talk to each other through well-defined interfaces. In general, there should be no reason for the interface to change just because an internal implementation detail changed. So other components shouldn't have to know about it.

I hope the reason why orthogonality reduces development time has become clear at this point. Changes applied to a system built with multiple cohesive components (meaning, self-contained, independent, and with a single clear purpose) are much faster and easier to test and understand. On the other hand, changes to a non-orthogonal system compound in a way that leads to confusion and slow iterations. The more components you add, the more unexpected side effects spread throughout the code, making the development workflow cumbersome.