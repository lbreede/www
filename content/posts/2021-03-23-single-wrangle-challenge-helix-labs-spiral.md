+++
date = 2021-03-23
title = 'Single Wrangle Challenge: Helix / Labs Spiral'
+++
Sometimes when I'm bored, waiting for renders or just have nothing better to do, I open an empty Houdini scene, put down a Detail Wrangle and see what I can create from scratch with just mathematics (and VEX of course). Some of the things I create the most are circles, waves and helices (plural of helix — yes, I had to look that up).

After the implementation of the [SideFX Labs](https://www.sidefx.com/products/sidefx-labs/) tools, all of the sudden, there was a definitive helix node called Labs Spiral with a bunch of nifty parameters to play with. For example, two sets of radii (that one I knew), a height parameter, and a helix count for multiple helices evenly rotated to create more complex shapes.

Since I now had a baseline of parameters a helix tool was supposed have, I got to work rebuilding it in one single wrangle. Half way through writing it, I realized that, since it's a Labs node, I could probably dive inside and see how it was built. And sure enough, I could, and I was even happier when I saw that it was built with eight nodes, six of them being high-level ones, e.g. *Line, Transform, Copy and Transform*, etc.

Here is the final result:

![](/helix_demo.gif)

It's not quite perfect to be completely honest.

My rotation and scale are in the wrong transform order, meaning I'm rotating first, then scaling, not vice versa, which is a problem for Future-Lennart to fix.

Furthermore, there is a Resample inside the Labs version which creates nicer spacing between points, but I'd rather leave that to myself to append and counting this change as a win.

In the end, I draw my own line when to stop. And done is, as we all know, better than perfect.

Below is the code for anyone who is interested or stuck at their own approach.

It's basically two loops, one for the helix count and the other one for creating circles with different radii, therefore making a helix.

Please reach out for questions, your own approaches, or requests for the next Single Wrangle Challenge (SWC? Also taking series name requests.).

Code edits:

2021-03-25: instead of creating a polyline on every j-iteration between the previous and the current point, I store all created points in an array and create the polyline at the end using that array. Resulting in a single primitive per spiral.

```c
vector2 radius = chu("radius");
float height = ch("height");
float loops = ch("loops");
        
float rotation = ch("rotation");
vector2 scale = chu("scale");

int points = chi("points");
int helix_count = chi("helix_count");

int pt;
float u, v, n, tau = PI * 2, rad, x, y, z;
vector pos;

for (int i = 0; i < helix_count; i++)
{
    u = float(i + 1) / float(helix_count);
    int pts[];
    for (int j = 0; j < points; j++)
    {
        v = float(j) / float(points - 1);
        n = tau * v * loops + tau * (-rotation / 360) + tau * u;
        
        rad = lerp(radius[0], radius[1], v);
        
        x = cos(n) * rad * scale[0];
        y = height * v;
        z = sin(n) * rad * scale[1];
        
        pos = set(x, y, z);
        
        pt = addpoint(0, pos); 
        
        append(pts, pt);
    }
    addprim(0, "polyline", pts);
}
```
