+++
title = "A ladder, a box and a wall"
description = "What could go wrong?"
tags = [
    "maths",
	"problems",
	"riddles"
]
aliases = [
    "/2015/06/30/ladder/"
]
date = 2015-06-30T19:31:00Z
author = "Yolan"
math = true
+++

I was recently reading a newspaper in the train and there was a little math riddle, I thought "how funny, that's gonna be easy, let's do it" and yet...

The problem goes as follow :
in a barn, there is a 1 meter cubic box against a wall and a 4 meter ladder is leaning against the wall, touching the box at its corner. Here is a picture :
<center>
![Illustration of the situation](/images/ladder-box.png "A ladder against a wall that touches a box.")
</center>

So, the big triangle has a hypotenuse $$FE$$ of $$4$$, the square $$ABDC$$ has sides of length $$1$$ and is basically "insquared" at the right angle, i.e. $$D\in \overline{FE}$$.
The question is "what is the length of the biggest cathetus", here $$AF$$.

So far, no problem.

Now here are my solutions:

 - By Thales' intercept theorem, $$\frac{FB}{FA}=\frac{BD}{AE}$$, by hypothesis, $$FB=FA-1$$ and $$BD=1$$. Now by Pythagoras, $$FA^2+AE^2=FE^2$$; by hypothesis, $$FE=4$$, so we end up with a system of equations, letting $$h=FA, d=AE$$: 
 
$$
 \begin{aligned} \frac{h-1}{h}&=\frac{1}{d} \\[2ex] h^2+d^2&=4^2  \end{aligned} 
$$

Which solves (removing 3 non-relevant solutions) into $$d \cong 1.3622$$ and $$h \cong 3.76091$$.
 - Now, if I consider the "function" of the line : $$f(x)=\frac{-h}{d}x+h$$, I know that $$f(1)=1$$ and I end up with Pythagoras with the system :
 
 $$ \begin{aligned} \frac{-h}{d}+h&=1 \\[2ex] h^2+d^2&=4^2  \end{aligned} $$
 
 it solves again into the same, again removing 3 non-relevant solutions

Okay, this means that using Pythagoras is no good since it ends up giving a quartic equation (4 answers, of which 3 are "non-relevant"). 

 - Now if I consider the length of the arc $$f(x)$$ between $$0$$ and $$d$$ it has to be $$4$$ and again $$f(1)=1$$ I end up with the system:
 
$$ 
\begin{aligned} \frac{-h}{d}+h&=1 \\ \int_0^d \sqrt{1+(f'(x))^2} dx &=\int_0^d \sqrt{1+\left(\frac{-h}{d}\right)^2} dx = d \sqrt{1+\left(\frac{-h}{d}\right)^2}   \end{aligned} 
$$

Which solves again into the same answers, but this time removing only 2 non-relevant solutions (i.e. it gives a cubic equation instead of a quartic).

I tried also using the areas and the smaller trangles $$FAD$$ and $$AED$$ for example : 

$$\frac{h \cdot d}{2} = \frac{h\cdot 1}{2}+\frac{d\cdot 1}{2}$$


Yet I wasn't able to get to any "hand solvable" solution : if I were able to bring it down to some quadratic equation, that would be nice, since it is a common assumption, here, that everybody has seen the "general formula for solving quadratic equations" in school and so would be able to solve this, I may then see how it is seen as a funny riddle in the newspaper. 

My best trial, with "just" a cubic equation, is way too complicated for the normal readers of this newspaper, so it's bugging me.

So, I [asked around](https://math.stackexchange.com/q/1344991/93584)! 
And [somebody proposed me](https://math.stackexchange.com/a/1345007/93584) the following solution, using a clever variable change:

Let $$|FB|=x$$. By similarity of triangles we then have $$|CE|=1/x$$. Pythagoras thus gives

$$
4=\sqrt{x^2+1}+\sqrt{1+(1/x)^2}=\sqrt{x^2+1}(1+\frac1x).
$$

Squaring this gives us

$$
16=(x^2+1)(1+\frac2x+\frac1{x^2}),
$$

but he preferred to move one factor $$x$$ from the former factor on the right hand side to the latter, so

$$
16=(x+\frac1x)(x+2+\frac1x).
$$

Almost there: write $$u=x+1/x$$. We can solve $$u$$ from the quadratic

$$
16=u(u+2),
$$

and then solve for $$x$$ from the equation 

$$
x+\frac1x=u.
$$
