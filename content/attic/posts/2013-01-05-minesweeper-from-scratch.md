---
layout: post
title: Winter Break Project - Building Minesweeper from Scratch
excerpt: "A simple project based tour of language features"
tags: [minesweeper, python, tkinter, tk]
date: 2013-01-05
comments: true
---

Code: [https://github.com/dmarx/mines](https://github.com/dmarx/mines)

I don't take a ton of vacation from my work, but luckily I don't really need to over the winter: my employer gives us christmas through new years off, which comes out to around 10 days vacation. I had planned to do some travelling, but instead tagged along with my girlfriend to do some family stuff for new years. Since I was in town for most of the break, I spent most of my time just chilling (which, frankly, was awesome) and coding.

I've been programming for a few years now, both for fun and for work, but my programs usually are either scripts or backend (database) programs. I never got around to learning GUI programming, so I decided to use this opportunity to build a GUI program. I had recommended some minesweeper related challenges to a friend who's learning programming, so i had minesweeper on the brain. I ended up coding the main game out of boredom one day, and decided to slap a GUI on top of it. 

Whenever I've seen GUI tutorials online, for simplicity sake they combine the main program code with the GUI code. I guess this makes sense for a quick tutorial, but my understanding is that this is pretty weak GUI programming. Generally, you want to separate the "model", "controller" and "view" components.

For my application, my model was comprised of a board class which is a container for tile class objects I also defined. The controller was a "game" class which included functions like "click" and "flip_flag." I wanted the game component (the model and controller) to operate just fine in isolation from the GUI, so my Game class also has a play() method for playing minesweeper in the terminal. Having built the working game components, I could now slap a GUI on top in a separate "view" component.

The most common GUI toolkits for python are wx, qt and tk. Tk is built into python as Tkinter, has a repuation for being simple to use, has plenty of documentation, and has the added benefit of taking advantage of the native OS's aesthetics, so I decided to go with Tkinter.

Now, although there are lots of Tkinter tutorials, there doesn't seem to be a ton of consistency in how people use Tkinter. In general, the main GUI is defined as a class called "App." But sometimes the App class inherits from the Tk class, and sometimes it inherits from the Frame class. I started using an example that had my App inheriting from Frame, and this worked fine. I later wanted to implement a feature where the buttons would be hidden when the game was paused, and I found that this required a lot of changes if I kept it inheriting from the Frame class. I changed my app to inherit from the Tk class and it was a lot easier to add that feature. I think that inheritance makes more sense intuitively as well.

Anyway, as an exercise, I strongly recommend programming minesweeper. Next time I need to learn a new language, I plan to use programming minesweeper as a comprehensive exercise. To accomplish the task takes using several primitive types and functions, defining classes, using threads (for the timer), and of course the GUI.



