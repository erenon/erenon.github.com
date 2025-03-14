---
title: "To hang an Icon"
date: 2025-03-14
---

I wanted to hang a picture, in a way that it doesn't touch the wall directly,
using a protruding hanger. If the hanger is difficult to observe, it makes
a nice floating effect.

I couldn't find a "µ" shaped piece of metal, and I don't have right drills for metal either,
so I decided to 3D-print the appropriate hanger. This would have been my first time doing this.

First, I needed a 3D modelling software, that is free and Linux compatible.
From the [Arch Linux offering](https://wiki.archlinux.org/title/List_of_applications/Science#Computer-aided_design),
I quickly selected [OpenSCAD](https://openscad.org/).

OpenSCAD is a wonderful software - instead of moving the model elements with a mouse,
it lets you describe the model using code, something I'm familiar with.
It is lightweight, with a clear, responsive UI.
By skimming the [wiki](https://en.wikibooks.org/wiki/OpenSCAD_User_Manual/Primitive_Solids#cube) and looking at examples,
I was able to create a simple model in less than 50 minutes. 
A testament of a great piece of software indeed.

![Screenshot of OpenSCAD, showing the editor, the rendered model, and the render status](/img/openscad.png)

The [code producing this model](https://gist.github.com/erenon/9788cdbe3c79128a117406eb3c7af36f) is short and simple.
The parametric nature allows quick changes, e.g: of thickness or depth.
After exporting the model to [.stl](https://en.wikipedia.org/wiki/STL_(file_format)),
I asked a friend to print it (thanks!). The Prusa machine did a beautiful job:

![The 3D-printed icon hanger, with a lego figure to scale](/img/icon-pin.jpg)

Finally, after a bit of measuring and drilling, the final composition:
modern and antique co-existing in perfect harmony.

![A monitor and the Trinity icon by Andrei Rublev side by side](/img/icon-and-screen.jpg)

