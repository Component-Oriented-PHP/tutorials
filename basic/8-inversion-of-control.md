---
layout: default
previous: 7-templates
---

# Inversion of Control (IoC): Service Locator and Dependency Injection

I am going to explain this IoC concept in a practical way. To do this, let's assume some idiot came to you and said twig is better than platesphp. You got in the mood to replace our existing platesphp with twig. You want to replace the existing templating engine with a new one. So, you went and replaced Plates Engine with Twig Loader in the PlatesRenderer class. Now, we did not need to change anything in the controllers so that's a plus for having a separate view rendering class.

However, what happens is your team lead later decides that they want to bring back platesphp. So now, you need to redo the PlatesRenderer class. However, what if you kept PlatesRenderer as it is and created a separate TwigRenderer class to handle rendering with Twig? Now you have two renderer classes, but your controllers are still hardcoded to use PlatesRenderer. To switch to TwigRenderer, you'd have to go into every single controller and change `new PlatesRenderer()` to `new TwigRenderer()`. If you have 50 controllers, that's 50 places to change. That's a maintenance nightmare.

Even worse, what if you want to support both templating engines and let users choose? Or what if you want to switch between them based on configuration? Our current approach makes this nearly impossible without major code changes.

Now this is where IoC design philosophy comes into the drama. This philosophy simply states that classes should not create their own dependencies; instead, dependencies should be provided to them from the outside.

That is, instead of a controller creating its own templating engine, the templating engine should be "injected" into the controller from somewhere else (where from? Be patient. I will explain). This way, the controller doesn't need to know or care about which specific templating engine is being used - it just works with whatever is given to it.

## Contracts (Interfaces)

You need to be aware of interfaces before I can go on. 