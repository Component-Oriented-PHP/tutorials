# So, You Wanna Build PHP Apps Without a Full-Blown Framework? (The Component Oriented Way!)

_TLDR: These tutorials teach intermediate to advanced PHP developers how to build applications using standalone Composer packages instead of full frameworks. Learn component-oriented architecture by assembling existing libraries into cohesive applications. Available tutorials: Basic (foundation concepts), Refactoring spaghetti code (implementing what we learned in basic tutorial) and Advanced (complex implementations). Requires solid PHP fundamentals and OOP knowledge._

You can read the tutorials in github repo directly or through:

- <https://component-oriented-php.github.io/tutorials/>

---

Alright, let's get one thing straight: if you're brand new to PHP, still figuring out what a `<?php` tag does, or if "Object-Oriented Programming" sounds like a cult you accidentally joined, this tutorial series might just chew you up and spit you out. (No hard feelings, we all start somewhere. Go get your PHP basics down, then come back. We'll have tea... or something stronger.)

In production, use established frameworks unless you have specific constraints (that frameworks can't handle) or learning goals. Use any framework you want - Symfony, Laravel, Yii, CI4, Laminas, Mezzio or any other that you and your team feel like using. It's rarely an issue in good hands.

**However**, _I wanted to give Patrick's original [No-Framework tutorial](https://github.com/PatrickLouys/no-framework-tutorial) a modern update, so I created this for anyone interested in learning "PHP application development without frameworks, by glueing together existing composer packages". This tutorial will be your guide if you ever feel the urge to move out of a framework or refactor legacy apps._

I use this approach in most of my personal projects and Symfony/CI4 for my business ones.

Available tutorials under the repo:

1. [Basics Tutorial](./basic/) - for noobs (or everyone)

2. [Refactoring Tutorial](./refactoring/) - we will refactor a spaghetti code into a modern one

3. [Advanced Tutorial](./advanced/) - for experts/intermediate players or the ones who have been through the Basic one (this one is so heart-wrenching that three developers have already rage-quit and gone back to WordPress. You've been warned; just kidding, LOL).

4. [Creating a static site generator](./static-site-generator/) - Creating a static site generator - a complete guide to build a CLI-powered static site generator by combining existing composer packages.

And one last thing... it is not an easy tutorial to follow if you are bad at PHP (doesn't matter if you're good at frameworks). People get so much into the framework ecosystem that they forget they are PHP devs, not framework devs. So, if you're not good at PHP, you would need to keep your eyes and mind open... because the tutorial is not easy (though I have made all possible attempts to keep it easy to follow; I remember having difficulty following Patrick's original tutorial, so I added this).

So, go on, read the next chapter/tutorial and comment/[discuss anything](https://github.com/orgs/Component-Oriented-PHP/discussions) once you're through the tutorial(s).
