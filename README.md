# Interfacing React Components with Angular Applications

There's been talk lately of using React as the view within Angular's MVC architecture. Angular, as we all know, uses dirty checking. As I'll touch on later, it accepts the fact of (minor) performance loss to gain the great 2 way data binding it has. React, on the other hand, uses a virtual DOM and only renders the difference. This results in very fast performance.

So, how do we leverage React's performance from our Angular application? Can we retain 2 way data flow? And just how significant is the performance increase?

The nrg module and demo code can be found [over on my GitHub](https://github.com/patrickmarabeas/nrg.js).