+++
title = "Yao's Garbled Circuits and how to construct those"
description = "It's not even that hard!"
tags = [
    "crypto",
	"talk"
]
aliases = [
	"/2017/06/09/garbling_circuits/"
]
date = 2017-06-09T23:22:00Z
author = "Yolan"
math = true
+++

I recently answered a nice question about garbled circuits, and I wanted to share my explanations. So let us review how garbled circuits actually work and how we can construct some. I'll try to explain this from top to bottom:

The protocol
------------
Let Alice and Bob be willing to compute *securely* a function $$f(x,y)$$ ([for example](https://research.cyber.ee/~peeter/teaching/krprot08s/kodu2.html), it could be $$f(x,y)=\min(x,y)$$) while keeping their respective inputs $$x$$ and $$y$$ **secret**. 

In order to do so, they will first model the function $$f$$ as a Boolean circuit, which is possible since there exists a Boolean circuit $$C$$ that calculates the output of $$f$$ for any function $$f$$ with fixed size inputs [1]. 
However, the way such a modelling is performed can depend on the function and will not be further discussed here.

Then Alice will *garble* the Boolean circuit: 
for each wire $$W_i$$ of the circuit $$C$$, she randomly chooses two secret values $$w^0_i, w^1_i$$, where $$w^j_i$$ is the *garbled value* of the value $$j\in \{0,1\}$$ of the wire $$W_i$$. 
Note that $$w_i^j$$ cannot reveal $$j$$ in itself, so Alice has to keep track of $$i$$ and $$j$$. 
This has to be done for each and every input and output wires of every logical gates in the circuit, excepted for the circuit output gates which can be left ungarbled.

Then Alice also has to construct a *garbled truth table* (GTT) $$T_i$$ for each of the gates $$G_i$$ in $$C$$. 
Those tables have to be so that given garbled values along its set of input wires, $$T_i$$ will allow to recover the garbled output of this $$G_i$$ and **no other information**. 
This is achieved through encryption of the output values. 
I will detail further the garbling of gates below.

[![A logical AND gate, its labels and its truth table][img1]][img1]

Afterwards, Alice can translate each bit of her input into its corresponding garbled values on the circuit's input wires. 
And then she can send the&mdash;now garbled&mdash;circuit where each gate is replaced by its GTT to Bob with her encrypted input.

After Bob received the garbled circuit, since all input wires are encrypted and only Alice knows the mapping of the encrypted values and the real bits, Bob need to perform an ***oblivious transfer*** (I let you [look that one up](https://en.wikipedia.org/wiki/Oblivious_transfer), since it's just a building block of the protocol) with Alice for each of his input bits, so that Alice can inform him of what encrypted values correspond to his inputs bits with knowing what are his actual input bits.

So it means that for each input wire, Bob will choose one amongst the two random strings $$w_i^0, w_i^1$$ which correspond respectively to $$0$$ and $$1$$, but without knowing the content of the string he does not choose. 
And thanks to the properties of the *oblivious transfer* Alice cannot learn Bob's input.

Then Bob has all the needed values to compute the output of the circuit as I will discuss it below. Once he has done so, he can tell the output values to Alice.
So Bob was able to obtain the output of $$f$$ without revealing his input, nor knowing Alice's input, this means that Alice and Bob successfully simulated a trusted third party and performed thus secure MPC!


Garbling Logical Gates
----------------------
The notion of garbling logical gates and their truth table is crucial. 
Without loss of generality, I will only consider logical gates with two input wires and one output wire. 
As I quickly explained above, for a given gate $$G\in C$$ and its input wires $$W_0,W_1$$ and its output wire $$W$$, Alice had to choose six different random strings, $$w_0^0, w_0^1,w_1^0,w_1^1,w^0,w^1$$ she assigned to each values of the wires in a one-to-one mapping, where $$w_i^j$$ represents the random string assigned to the value $$j$$ of the wire $$W_i$$.

Then in order to garble the truth table of $$G$$ in a way which would not reveal any information given two input values $$w_0,w_1$$ excepted for its output value $$w$$, not even the type of the gate, Alice can encrypt the output values $$w^0,w^1$$ using the garbled input values as keys, using a given symmetric encryption scheme $$\mathbb{E}$$. 
I use the notation $$\mathbb{E}_{k_0,k_1}(x)$$ $$=\mathbb{E}_{k_0}$$ $$(\mathbb{E}_{k_1}(x)) $$ to denote encrypting twice with two given keys $$k_0,k_1$$. 
As an example, let us encrypt the truth table of the AND gate from Figure 1:  

$$ \begin{array}{c|c|c||c}W_0 & W_1 & W & \text{Garbled value} \\   w_0^0 & w_1^0 & w^0 & \mathbb{E}_{w_0^0,w_1^0}(w^0) \\ w_0^0 & w_1^1 & w^0 & \mathbb{E}_{w_0^0,w_1^1}(w^0) \\ w_0^1 & w_1^0 & w^0 & \mathbb{E}_{w_0^1,w_1^0}(w^0)\\ w_0^1 & w_1^1 & w^1 & \mathbb{E}_{w_0^1,w_1^1}(w^1)\end{array} $$
 
The GTT $$T$$ of the gate $$G$$ is then simply the set $$\left\lbrace\mathbb{E}_{w_0^j,w_1^k}(w^{G(j,k)}) \mid j,k \in  \left\lbrace0,1\right\rbrace \right\rbrace$$ of the garbled values, where $$G(j,k)$$ corresponds to the output of the gate $$G$$ under inputs $$(j,k)$$.

Evaluation of a garbled circuit
--------------------------------
Once Bob has received the garbled circuit $$C$$ from Alice and obtained the garbled values of his input through multiple *oblivious transfers*, he can evaluate the circuit. 

It is important to realize that a garbled circuit differs from a regular Boolean circuit.
In a Boolean circuit, **semantic** and **syntax** are basically the same: we are assigning to each wire two possible *semantic* values, namely `True` or `False`, which we will *syntactically* denote as a signal with 1 or 0 values respectively. 

Those signals were public and the same signals were associated with every wire and anybody could tell from the signal what semantic value it held. 
*This changes in a garbled circuit*, since the **semantic** values of each *signal*&mdash;excepted for the circuit's output ones&mdash;are now secret and the *signals* are now varying from one wire to another wire.

Thus, in order to evaluate the circuit, for each gates $$G_i$$ of the circuit, Bob can try and decrypt the values in the associated truth table $$T_i$$ using the gate's input values as keys. One of the entries in $$T_i$$ will then decrypt into the output of the gate. 
Hence it seems necessary to have an oracle to confirm successful decryption of the entries of $$T_i$$, however a trick I'll detail right now called **permute-and-point** first used in [2] and then clearly explained in Phillip Rogoway's dissertation [3], allows to decide which entry of the GTT is to be decrypted given the garbled inputs, enabling faster computations and still preventing the evaluator to deduce anything from the order of the truth table's entries.

Permute-and-point
-----------------

This trick works as follows: for each input and output wires $$W_i$$, Alice will concatenate a random bit $$a\in \{0,1\}$$ at the end of its garbled value $$w_i^0$$ and concatenate its inverse $$b=\overline{a}=1-a$$ at the end of $$w_i^1$$. So it allows to associate each of the 4 permutations of 2 bits to one of the entry of the GTT, without having any correlation between the bits and the values of the ungarbled truth table. So Alice can simply order the GTT according to the natural ordering and give it to Bob which will then be able to deduce which entry he should decrypt on a given input.
To obtain a nice representation of this trick, those bits can be seen as a pair of *colors*, as I depict them in Figure 2, in which we see how the truth table is modified to take into account this method.

[![The permute-and-point trick][img2]][img2]

Those modifications allow Bob to simply decrypt the entry whose index corresponds with the *colors* associated to its input wires and thus to obtain the output wire's value and its *color*, allowing him to further evaluate the circuit.

----------

Example evaluation with permute-and-point
-----------------------------------------

[![A circuit made of one AND gate and one OR gate and one XOR gate with their respective GTT and their labeled wires randomly colored. The bold values are the values used to compute the example below.][img3]][img3]


Let us see how one could evaluate the garbled circuit represented in Figure 3 using the *permute-and-point* method we discussed above. 
Let us assume that the semantic input values of $$(W_1,W_2,W_3,W_4)$$ are $$(0,0,1,0)$$, which means that the actual garbled input is $$({\color{green}\bullet}{}w_1^0,{\color{red}\bullet}{}w_2^0,{\color{red}\bullet}{}w_3^1,{\color{green}\bullet}{}w_4^0)$$ where the $$w_j^i$$ are the random values that Alice chose when she garbled the circuit, as seen above. 
Let us also assume that Alice already provided her garbled input, say $$(W_1,W_3)$$, and that Bob already obtained his garbled input $$(W_2,W_4)$$ from Alice through two applications of *oblivious transfer* as described in the protocol part.

So Bob will begin by evaluating first the AND gate using the input $$({\color{green}\bullet}{}w_1^0,{\color{red}\bullet}{}w_2^0)$$, since it has the colors $$ \color{green}\bullet\color{red}\bullet$$ 
he will try and decrypt the third entry of the GGT of the AND gate, which works and thus provide him with the garbled value $${\color{green}\bullet}{}w_5^0$$.

Then he can continue his evaluation with the second gate, which is the OR gate. He looks at its input $$({\color{red}\bullet}{}w_3^1,{\color{green}\bullet}{}w_4^0)$$ and try to decrypt the entry corresponding to $$ \color{red}\bullet\color{green}\bullet$$ with the keys $$(w_3^1,w_4^0)$$, it decrypt into the garbled value $${\color{green}\bullet}{}w_6^1$$.

He can now decrypt the final XOR gate using the computed input $$({\color{green}\bullet}{}w_5^0,{\color{green}\bullet}{}w_6^1)$$, thus decrypting the $$\color{green}\bullet\color{green}\bullet$$ entry which provide him with the final result: "1".

And Bob does not know what was Alice's input, he just knows the final output "1" and the randomly generated strings $$w_1^0,w_2^0,w_3^1,w_4^0,w_5^0,w_6^1$$. He can still for example deduce from the circuit that the semantic values of $$w_5^0$$ and $$w_6^1$$ are opposed, however it does not allow him to reverse the garbled circuit up to Alice's input values. There are circuits where no privacy is assured, like for example a circuit computing the sum of the inputs, however in this example case, Bob&mdash;knowing his semantic input values&mdash;can simply restrict Alice's inputs to a subset of the possible inputs but cannot uniquely determine the actual Alice's input values.

----------

Example: modelling the min function
-----------------------------------
The fastest way to model a boolean function is to draw its truth table, so in our case, we have 4 bits inputs, with the numbers $$a$$ and $$b$$ being 2-bits numbers $$a_1a_2$$ and $$b_1b_2$$ respectively, and we want as output the minimum $$c$$ of $$a$$ and $$b$$, where $$c$$ is constituted of two bits: $$c_1c_2$$.

So we can draw the truth table:


$$\begin{array}{c|c||c|c||c|c}
a_1 & a_2 & b_1 & b_2 & c_1 & c_2 \\  
0 &	0 &	0 &	0 &	0 &	0 \\
0 &	0 &	0 &	1 &	0 &	0 \\
0 &	0 &	1 &	0 &	0 &	0 \\
0 &	0 &	1 &	1 &	0 &	0 \\
0 &	1 &	0 &	0 &	0 &	0 \\
0 &	1 &	0 &	1 &	0 &	1 \\
0 &	1 &	1 &	0 &	0 &	1 \\
0 &	1 &	1 &	1 &	0 &	1 \\
1 &	0 &	0 &	0 &	0 &	0 \\
1 &	0 &	0 &	1 &	0 &	1 \\
1 &	0 &	1 &	0 &	1 &	0 \\
1 &	0 &	1 &	1 &	1 &	0 \\
1 &	1 &	0 &	0 &	0 &	0 \\
1 &	1 &	0 &	1 &	0 &	1 \\
1 &	1 &	1 &	0 &	1 &	0 \\
1 &	1 &	1 &	1 &	1 &	1
\end{array}$$

And now we can model our boolean circuit by taking the `true` values in the results and represent them as being boolean expressions:

For $$c_1$$:

$$
\begin{array}{c|c||c|c||c|c}
a_1 & a_2 & b_1 & b_2 & c_1 & {bool. expr.} \\  
1 &	0 &	1 &	0 &	1 & a_1 \land \overline{a_2}\land b_1 \land \overline{b_2}	 \\
1 &	0 &	1 &	1 &	1 &	a_1 \land \overline{a_2}\land b_1 \land {b_2} \\
1 &	1 &	1 &	0 &	1 &	a_1 \land {a_2}\land b_1 \land \overline{b_2} \\
1 &	1 &	1 &	1 &	1 &	a_1 \land {a_2}\land b_1 \land {b_2}
\end{array}
$$

And for $$c_2$$:

$$
\begin{array}{c|c||c|c||c|c}
a_1 & a_2 & b_1 & b_2 & c_2 & {bool. expr.}  \\  
0 &	1 &	0 &	1 &	1 & \overline{a_1} \land {a_2}\land \overline{b_1} \land {b_2}\\
0 &	1 &	1 &	0 &	1 & \overline{a_1} \land {a_2}\land b_1 \land \overline{b_2}\\
0 &	1 &	1 &	1 &	1 & \overline{a_1} \land {a_2}\land b_1 \land {b_2}\\
1 &	0 &	0 &	1 &	1 & a_1 \land \overline{a_2}\land \overline{b_1} \land {b_2}\\
1 &	1 &	0 &	1 &	1 & a_1 \land {a_2}\land \overline{b_1} \land {b_2}\\
1 &	1 &	1 &	1 &	1 & a_1 \land {a_2}\land b_1 \land {b_2}
\end{array}
$$

Now, we can express $$c_1$$ and $$c_2$$ as the OR of all their respective boolean expression, this method is called the boolean Sum-of-Product:

$$
\begin{aligned}
c_1= (a_1\land \overline{a_2}\land b_1 \land \overline{b_2})	
\lor(a_1\land \overline{a_2}\land b_1 \land {b_2}) \\
\lor(a_1\land {a_2}\land b_1 \land \overline{b_2}) 
\lor(a_1\land {a_2}\land b_1 \land {b_2}) 
\end{aligned}
$$

And for $$c_2$$:

$$
\begin{aligned}
 c_2 = (\overline{a_1} \land {a_2}\land \overline{b_1} \land {b_2})
\lor(\overline{a_1} \land {a_2}\land b_1 \land \overline{b_2})\\
\lor(\overline{a_1} \land {a_2}\land b_1 \land {b_2})
\lor(a_1 \land \overline{a_2}\land \overline{b_1} \land {b_2})\\
\lor(a_1 \land {a_2}\land \overline{b_1} \land {b_2})
\lor(a_1 \land {a_2}\land b_1 \land {b_2})\\
\end{aligned}
$$

And we have a possible circuit (represented as boolean expressions) for our $$\min$$ function.

But those expression are really ugly and we want to simplify them, it is possible to convert them into nicer forms, like [DNF](https://en.wikipedia.org/wiki/Disjunctive_normal_form) (this is where [Karnaugh maps](https://en.wikipedia.org/wiki/Karnaugh_map) can help). But I won't cover this topic today.

For instance, $$c_1$$ can be rewritten as: $$c_1=a_1 \land b_1$$ (actually we can see this directly from the truth table) and $$c_2$$ can be rewritten as:
$$c_2 = (a_1 \land \overline{b_1} \land b_2) \lor (\overline{a_1} \land a_2 \land b_1) \lor (a_2 \land b_2)$$

I haven't much resources to refer you to there, but a quick search led me to [this page](https://www.allaboutcircuits.com/textbook/digital/chpt-7/converting-truth-tables-boolean-expressions/) and [that one, which even has a graphical representation](http://www.32x8.com/var4.html) of the circuit, once given the truth table, nice!    

----------

References
-----------
[1] Goldwasser, S., Micali, S., & Wigderson, A. (1987). How to play any mental game, or a completeness theorem for protocols with an honest majority. In Proc. of the Nienteenth Annual ACM STOC (Vol. 87, pp. 218-229). [[PDF]](https://www.researchgate.net/profile/Oded_Goldreich/publication/234778924_How_to_play_ANY_mental_game/links/0deec5232112523fc5000000.pdf)

[2] Beaver, D., Micali, S., & Rogaway, P. (1990, April). The round complexity of secure protocols. In Proceedings of the twenty-second annual ACM symposium on Theory of computing (pp. 503-513). ACM. [[PDF]](https://pdfs.semanticscholar.org/f71d/b4d70d4cc9e931a63dde7a6db8dad10a61a0.pdf)

[3] Rogaway, P., & Micali, S. (1991). The round complexity of secure protocols (Doctoral dissertation, Massachusetts Institute of Technology, Dept. of Electrical Engineering and Computer Science). [[PDF]](http://mame.myds.me/bitsavers/pdf/mit/lcs/tr/MIT-LCS-TR-503.pdf)

PS: I have shared these images and their source TeX code on [TikZ for Cryptographers](https://www.iacr.org/authors/tikz/) if you don't know this project, check it out, it's awesome.

  [img1]: //romailler.ch/images/garbled_1.png


  [img2]: //romailler.ch/images/garbled_2.png


  [img3]: //romailler.ch/images/garbled_3.png
