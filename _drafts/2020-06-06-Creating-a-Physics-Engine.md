---
title: Creating a Physics Engine Part 1
---

Today, I am currently creating a basic physics engine that will be used for any future projects that I may require the usage of. Why? Firstly, it touches on two subjects that I absolutely adore, Physics and Computer Science. Secondly, it's fun! Along the way, I will also be giving some references I was using when building this project. A large portion of it is from [Randy Paul's How to Create a Custom 2D Physics Engine](https://gamedevelopment.tutsplus.com/tutorials/how-to-create-a-custom-2d-physics-engine-the-basics-and-impulse-resolution--gamedev-6331/). If you really want to dive deep into the details, check out his guide! This post is primarily documenting what I have learnt from this guide and other sources as well. My goal is to make it REALLY easy for anyone to at least understand how the physics engine tries to imitate reality.

Most of the [code I have written](https://github.com/Assyarul/Particle-System) is primarily in JavaScript using [p5.js](https://p5js.org/) to display the scene so that I can show the engine working easily in a website. I suggest looking up on p5.js There are also Processing libraries for Java and Python. Setting up the visual environment is as easy as creating a setup() function and draw() function in a js file.
{% highlight javascript %}
function setup() {
    //all the necessary setting up for the environment goes here.
}

draw() {
    //updating positions of the objects and rendering it goes here at a frame by frame basis.
}
{% endhighlight %}

A physics engine typically has two core components.

1. A **Dynamics Solver** responsible for handling the forces and thus the motion of the objects in the simulation/scene.

2. A **Collision Solver** that handles the detection and the resolution of any collisions between objects. 

The objects in the simulation also holds vital data that is needed for the engine to do its job, such as its position, velocity and mass. The p5.js library also has a createVector function that enables us to create a Vector object.

{% highlight javascript %}
class Object{
    constructor(x, y, mass){
        this.position = createVector(x,y); 
        this.velocity = createVector(0,0);
        this.force = createVector(0,0);
        this.mass = mass;
    }
}
{% endhighlight %}

It is important to highlight that position and velocity are vectors since we are dealing with 2D space here. The object also contains its own force component as it will make it easy for the engine to just exert different forces onto the Object and it will just calculate the sum of the forces to determine the final acceleration onto the object at the end.

# Dynamics Solver
I personally had a blast learning about the implemention of solving the dynamics of a physics system since it involves in Newton's Three Laws of Motion.
The implementation of the First Law of Newton, is trivial. As long as no forces are on an object, its velocity should remain the same.

The tricky part comes with the Second Law of Motion, 

$$ F = ma = m \frac{dv}{dt} $$

$$ \frac{dv}{dt} = F\times \frac{1}{m} $$

What's most important is to obtain the change in velocity (dv) so that we can update our velocity and position of our object for the next frame. To obtain it, we have to integrate the equation above with respect to time. Computing integrations in a computer however requires *a little* bit of work. I recommend that you explore the various ways to do numerical integrations. The tutorial I mentioned also gave a good reference for [this](https://gafferongames.com/post/integration_basics/). 

The simplest way to do integrations on a computer is via Euler's method of integration. In essence, with respect to Newton's Second Law of Motion, all I am doing is keeping the dv on the right hand side of the equation and the rest on the left. This is a very brief explanation of how it works and will explain it in the context of the program later on.

![Explaining Euler's method](\assets\images\blog\2020-05-06-Introduction-To-Me\graph.svg)

$$ dv = F\times \frac{1}{m} \times dt $$

To find the change in velocity (**dv**), multiply the acceleration by **dt**. What **dt** represents is a timestep and what this physically represents in the engine is the time taken between two 'updateObject()' call of a particular object. This is not really true for now but we will make it so later on.

For example at t=0s, a 1kg ball is 0 metres away from me. I give it a shove, exerting an initial force of 1 Newtons. 

![At time t=0s](\assets\images\blog\2020-05-06-Introduction-To-Me\time0.svg)
 
If I were to fix dt = 0.1 second ( This would mean it is as though the physics engine is running at '10 fps'), At t=0.1 seconds , the ball will now have a velocity = 0.1 m/s and being 0.1 m away from me. 

![At time t=0.1s](\assets\images\blog\2020-05-06-Introduction-To-Me\time1.svg)

If you were in the engine, there was no time passed between t=0s and t=0.1s. The velocity didn't 'gradually' increase to 0.1m/s. It is immediately setted to 0.1m/s. This also applies to the position of the object. You can argue that in this physics engine, the object looks like it was 'teleporting' at small increments at a time. Very cool.

Reasonably, if you think of it in the context of the simulation, a smaller dt would mean that your engine will be more accurate in its simulation. A smaller dt would mean smaller changes in the object's velocity and subsequently its position. Slowly, it starts to approximate much closer to the continuous nature of motion we have in the real world ( I would argue that if you set dt to be the smallest unit of time, i.e Planck's time, maybe it would be no different from reality?). A smaller dt however would mean that it would be that much more computationally expensive to run the simulation as a whole as a lot more updates has to be made to objects within a fixed time period. A compromise has to be made between accuracy and speed.

{% highlight javascript %}
class Object{
    constructor(x, y, mass){
        this.position = createVector(x,y); 
        this.velocity = createVector(0,0);
        this.force = createVector(0,0);
        this.mass = mass;
    }
    updateObject(dt) {
        this.velocity.add(this.totalforce.copy().mult(1/this.mass).mult(dt)); // v += dv where dv = force x 1/m x dt
        this.position.add(this.velocity.copy().mult(dt))  // x += dx where dx = v x dt
    }
}
{% endhighlight %}

If you are confused about the methods used in the code above, the class methods of p5.Vector can be searched easily through the p5.js [documentation](https://p5js.org/reference/#/p5.Vector).

Remember what I said about **dt** being the time taken between two updateObject() calls for an object? In a typical game environment, the fps (frames per second) will always change, often depending on the workload of the processor/GPU. If we were to set dt to a fixed time and run updateObjects for each frame,

{% highlight javascript %}
draw() {
    //Note this is not the actual code from the Engine I've made. Just to make it simple to understand.
    // Update all the objects in the simulation
    objectList.forEach((object)=>object.updateObject(dt));
}
{% endhighlight %}

 This is no longer true. The physics engine is treating as though dt has passed but the real time taken is actually 1/current_fps. This will cause the time in the engine not reflect time taken in reality. Ok, why not make dt to be the actual time taken between two frames?

 {% highlight javascript %}
draw() {
    dt = deltaTime; //deltaTime is a system variable in p5.js that calculates time taken between two frames
    // Update all the objects in the simulation
    objectList.forEach((object)=>object.updateObject(dt));
}
{% endhighlight %}
 
 Here, we are setting the processing of the environment to be **frame-rate dependent**, which also has its own cans of worms. This means that the game will 'run' much slower at 30 FPS compared to 60 FPS! The game will appear to move as though it is Keanu Reeves dodging bullets in the Matrix.

![Dodging bullets](https://thumbs.gfycat.com/JovialEarlyEastrussiancoursinghounds-size_restricted.gif)

Credits: [https://gfycat.com/jovialearlyeastrussiancoursinghounds-bullet-matrix](https://gfycat.com/jovialearlyeastrussiancoursinghounds-bullet-matrix)

There are also many more problems that comes with this such as objects phasing through other objects if the fps is low enough that no collision was detected. An easy fix to this is to update the physics only when a certain time has elapsed.





