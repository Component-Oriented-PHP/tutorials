---
previous: 7-templates
---

# Inversion of Control (IoC): Service Locator and Dependency Injection

I am going to explain this IoC concept in a practical way. To do this, ...

Now this is where IoC design philosophy comes into the drama. This philosophy simply states that classes should not create their own dependencies; instead, dependencies should be provided to them from the outside.

That is, instead of a controller creating its own templating engine, the templating engine should be "injected" into the controller from somewhere else (where from? Be patient. I will explain). This way, the controller doesn't need to know or care about which specific templating engine is being used - it just works with whatever is given to it.
