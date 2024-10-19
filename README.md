
Search
Write
Sign up

Sign in



Powerflow analysis using Python
Francesco Bazzani
Francesco Bazzani

·
Follow

7 min read
·
Sep 13, 2022
14





Curious to know how network operators manage the Electrical System? Dive with me in the power-flow algorithm, used to determine AC power flows in electrical grids, implemented using python.

Electric power has enabled a strong technological advancement in our society, but it has a major flaw, however: at any instant, the electric power generated throughout the grid must be equal to the electric power absorbed throughout the grid. In essence, the generation must equal the load, including the energy transport losses, that is, the energy that is dissipated in the power cables due to the Joule effect. Hence the fundamental role of electrical planning (i.e. generation and load management), which uses Powerflow algorithms to verify the feasibility of a given generation/load condition.

There are generally two types of companies that make heavy use of the Powerflow algorithm: the TSOs (transmission system operators) that operate the High-Voltage networks (HV), and the DSOs (Distribution System Operators) that operate the medium- and low-voltage networks. Low-Voltage networks (LV) are the ones that, generally, come into our homes.
Every power grid operates at a nominal voltage rating. The LV grid nominal voltage, for the United States, is 110V, while for Italy, the country from which I write, it is 230V. In general, however, this is an indicative value; in fact, all the appliances we use every day have a variability of accepted values (e.g. 100V — 250V). Precisely because of the mathematical form of the equations, it is not possible to exchange power between two points in a power grid if they are at the same voltage (module and phase), and therefore each device must accept as input a range of voltage levels.

Image of a phone charger: the input voltage ranges from 100V to 240V
My phone charger datasheet: the input voltage ranges from 100V to 240V
As is well known all modern electrical systems use alternate (sinusoidal) current. I won’t explain the advantages of this system, nowadays mainly historical and economic. To look deeper into this topic: https://en.wikipedia.org/wiki/War_of_the_currents

The mathematical advantage of any sinusoidal quantity is that it can be expressed through a complex exponential. Every complex number can be described by 2 numbers: the module and the phase or, equivalently, the real and the imaginary part. With this algorithm, we want to calculate, for each node of the network, 2 complex numbers: the voltage and the complex power. The module of the voltage is the value that we would measure at the node using a voltmeter and has to be a value between the maximum and minimum limits of every device connected at that node.

The phase of the voltage is the “angular distance” between the node and the “reference node”, called also slack node. I will go deeper into this in the following. Could seem less important, but its value is responsible for the transits of the power between two connected nodes (or, looking at it from another perspective, the power flows between two nodes modify the “angular distance” between these nodes).

The real part of Apparent Power is the Active Power. In the image shown before, my phone charger has a maximum load power of 60W. The imaginary part of Apparent power is the reactive power. It’s a value that is not responsible for any “active” energy transport. Informally represents zero mean value energy. I could try to describe these quantities in more detail, but it would perhaps be of little use to pass the concept of what these quantities represent. Instead, this image usually describes the phenomenon very clearly.

image that represents the Apparent power. What we drink is the beer, the foam is only a subproduct of the beer.
What we drink is the beer, the foam is only a subproduct of the beer.
When we want to drink a beer we are forced also to drink the foam. The same with electrical power: to actively transfer power to a load is necessary to transfer Active and Reactive power to the load, even if, as described before, the Reactive Power transfer a zero-mean amount of energy at the load.

The equations that describe the power flows in a network are nonlinear functions of the network Loads, Generators, Topology, and Network parameters — resistance, inductance, conductance, and capacity and length of each line of the network. As discussed before, since the nonlinear characteristic of these equations, the only viable way to solve them is through iterative methods. In the following, I will describe the Newton — Raphson solution method. For each node, we have to 4 interesting quantities: Voltage, Angle, Active Power, and Reactive power. Depending on the node type, the imposed quantities vary.

The PV nodes are the generation nodes, in which large generation units emit the energy that they produce (nuclear plants, solar farms, coal plants, and so on…). Here the fixed quantities are the Active Power and the Absolute Value of the Voltage.

The PQ nodes are the load nodes. Is safe to suppose that the majority of the nodes of a network are PQ nodes. To make things more practical is possible to assume that every home is a PQ node. In a PQ node, the fixed values are the Active Power and the Reactive Power absorbed. The Voltage is not fixed and is therefore calculated accordingly. This brings us back to the initial argument, in which I pointed out that the nominal value was not the actual value of the voltage that can be measured in every socket in the house.

The last variety of nodes is the Slack node. I have not casually used the singular: for each network, there is only one slack node. The slack node has two basic tasks: to balance the energy flows, thus compensating for the energy dissipated in the transmission lines, and to be a reference for all other generation PV nodes. The Slack node fixes its Angle to 0 and its Absolute Value of the Voltage. The slack node must be a generating node, the purpose of which is to be a reference for all other voltages in the network: if, for example, we calculate that a node has a voltage amplitude of 218.5 V and a phase of 30 degrees, this means that the voltage will be 95% of the slack bus voltage (230V) and will be shifted by 30 degrees with respect to its voltage.

The implementation of the Algorithm

To implement this algorithm I have created a Grid Class in python.


Grid class constructor. Each Grid instance is constructed from a list of nodes and lines.
The constructor of the Grid object requires two lists: nodes and lines, made by Node and Lines objects. In the following i reported the implementation of these two objects.


Node object implementation. For each node is possible to fix the type (1=PQ, 2=PV, 3=Slack), the generated or absorbed (as a load) power, and the voltage magnitude and angle. These are the starting values used in the load-flow calculations.

Line object implementation. The constructor accepts the line terminals, both Node objects, and the electrical parameters of the line (using the pi-model).
The Newton — Raphson method allows linearizing of the original nonlinear system of equations. The implementation of this algorithm is made in the loadflow method of the Grid class. Our objective is to calculate the 2 floating variables, the ones that are not fixed by the type of the node — for instance, voltage magnitude and angle for a PQ node. Let’s walk through the code to understand what’s happening.

This method begins with initial guesses of all unknown variables. At the first iteration, these variables are set at the values fixed in the Node instances; this is made by the Node constructor, which fixes the self.vlf parameter to the nominal voltage.

Next, the algorithm solves the power balance equations using the most recent voltage angle and the magnitude value.



Power balance equations. For each node i the algorithm looks at all its surrounding N nodes.

Power balance calculations
The matrixes B and G take into account the electrical parameters of the lines surrounding the i-th node. If the line that connects the i-th with the k-th node is old and has “low electrical performances”, then the contribution of the power coming from the k-th node is low. It basically means that if your house is connected at two lines, one new and efficient, the other old and inefficient, most part of your consumption will come from the newer one.

The B and G matrixes are calculated by the create_matrix method of the Grid class.


The create_matrix method creates the B and G matrixes starting from the electrical parameters of the grid
After this step, the algorithm calculates the variation of the active and reactive power with respect to the precedent step.


This calculation is necessary because it allows us to calculate the variation of voltage angle and magnitude, using the Jacobian matrix.


Linearized relation between change in Active and Reactive power and change in Angle and Magnitude of the voltage

Jacobian calculation
Here is the implementation of the Jacobian calculation.


After calculating, and inverting, the Jacobian is possible to calculate the variation of the voltages and update their values.


If tol reaches a value below the fixed threshold (10^-5 in this example) the algorithm stops. It has converged!

After reaching convergence the loadflow method calls the calculateLf method. This is an auxiliary method that, given a viable solution to the Powerflo problem, calculates the Active and Reactive power flows for each line of the network.


Auxiliary method to calculate all the power flows in the network
The complete implementation of the algorithm can be found on my GitHub at this link.

What a journey! I am very happy to have shared this project! It combines my two passions: electrical engineering and computer science engineering.
In the comments let me know what you think and share the post on all social networks! Want to get in touch with me? You can contact me on LinkedIn, or write me an email.
Did you like the article? Consider buying me a coffee, you could bring a smile to a developer’s grey day :D

Francesco
Hi! I'm an electrical engineer and a computer science student. If you're here is probably because you read one of my…
www.buymeacoffee.com

Python
Electricity
Load Flow Analysis
14



Francesco Bazzani
Written by Francesco Bazzani
10 Followers
Electrical and Computer Science Engineer

Follow

More from Francesco Bazzani
Francesco Bazzani
Francesco Bazzani

Kaseya ransomware attack: a cyber kill chain analysis
This article presents an analysis of the 2021 Kaseya cyber-attack using the cyber kill chain framework. The kill chain outlines the stages…
Feb 27, 2023
Ransom note
See all from Francesco Bazzani
Recommended from Medium
Lambda Function in Python
NIRAJAN JHA
NIRAJAN JHA

Lambda Function in Python
A lambda function is an anonymous function (i.e. defined without a name) that can take any number of arguments but, unlike normal…
Apr 24
3
The resume that got a software engineer a $300,000 job at Google.
Alexander Nguyen
Alexander Nguyen

in

Level Up Coding

The resume that got a software engineer a $300,000 job at Google.
1-page. Well-formatted.

Jun 1
24K
485
Lists



Coding & Development
11 stories
·
854 saves



Predictive Modeling w/ Python
20 stories
·
1599 saves
Principal Component Analysis for ML
Time Series Analysis
deep learning cheatsheet for beginner
Practical Guides to Machine Learning
10 stories
·
1947 saves

AI-generated image of a cute tiny robot in the backdrop of ChatGPT’s logo

ChatGPT
21 stories
·
839 saves
5 AI Projects You Can Build This Weekend (with Python)
Shaw Talebi
Shaw Talebi

in

Towards Data Science

5 AI Projects You Can Build This Weekend (with Python)
From beginner-friendly to advanced

Oct 9
2.9K
47
I used OpenAI’s o1 model to develop a trading strategy. It is DESTROYING the market
Austin Starks
Austin Starks

in

DataDrivenInvestor

I used OpenAI’s o1 model to develop a trading strategy. It is DESTROYING the market
It literally took one try. I was shocked.

Sep 16
4.1K
115
Build a Beautiful Weather App with Flask and OpenWeather API: A Step-by-Step Guide
Kuldeepkumawat
Kuldeepkumawat

Build a Beautiful Weather App with Flask and OpenWeather API: A Step-by-Step Guide
Introduction:

Aug 9
Encryption and Decryption in Python Using OOP
Python Coding
Python Coding

Encryption and Decryption in Python Using OOP
Explanation:

Sep 4
62
1
See more recommendations
Help

Status

About

Careers

Press

Blog

Privacy

Terms

Text to speech

Teams
