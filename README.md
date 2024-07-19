# PLC-Crossy-Road
Play Crossy Road with Studio5000, Logix Emulate, and FactoryTalk View!

# How to play
When testing and debugging this project on my PC, I had communications set up as follows: in RSLinx Classic Lite --> configure drivers, I selected and added "Virtual Backplane (SoftLogix58xx USB)" with the default name in slot 1. In Logix Emulate, I added Emulate 5570, V30, to slot 0 of the virtual backplane. When this was done, Logix Emulate was as shown in Figure1.png. The communications in Crossy_Road.mer and Crossy_Road.api are configured accordingly by default, but of course you may change them for whatever controller you use. 

Download and extract CrossyCode.zip. Open Crossy_Road.ACD; download program to the controller and put in run mode. Open (load) and run Crossy_Road.mer. 

When you select a character and hit 'PLAY', control player movement with the four buttons on the buttom right-hand corner. (I know it is annoying; more on this later). 

# Setting parameters
You may want to adjust the game to fit your preferences. Crossy_Road.ACD has a few controller tags that make this easy.

**Scroll_Speed** sets the speed that the rows are shifted down the screen. 

**Lane_Density** sets the percentage of rows that are lanes (_Type=1_). **Lane_Density** = 1 is only lanes, and **Lane_Density** = 0  is no lanes.    

**Lane_Speed_Min** and **Lane_Speed_Max** set the minimum and maximum speeds of a new lane. When a row is randomized and it is a lane, its speed (in theory) is selected randomly from a uniform distribution between the minimum and maximum speed. 

**Lane_Car_Density_Min** and **Lane_Car_Density_Max** are analogous to **Lane_Speed_Min** and **Lane_Speed_Max**, but for the _Car_Density_ property. 

**Tree_Density** determines the proportion of trees that will be active in newly generated rows of type 0. **Tree_Density** = 1 is all tree, and **Tree_Density** = 0 is no trees. 

**Collision_Offset_Min** and **Collision_Offset_Max** are used to determine how easily a car hits the player. **Collision_Offset_Min** is the minimum offset required for an active car in the car column to the left of the player in the same row to hit the player. Setting it lower will extend the range the car kills the player in front of it. Likewise, **Collision_Offset_max** is the maximum offset required for an active car in the car column to the right of the player in the same row to hit the player. Setting it higher will extend the range the car kills the player behind it. 

**Grass_Strip** is the way the bottom three rows (see MainRoutine) are initialized. 

**Random_Mode** determines how random numbers are obtained. When equal to 0 (default) they are generated sequentially from a list (GenerateSequentialNumber). When equal to 1, they are equal to 1, they come from the wall clock time (GenerateRandomNumber). 

**Random_Number_Divisor** only applies when **Random_Mode** = 1. Setting this to a high value will result in more possible results from GenerateRandomNumber, but a higher period (ie. the numbers generated oscillate slower). 

**Sequential_Numbers** only applies when **Random_Mode** = 0. It is a list of the random numbers flipped through by GenerateSequentialNumber. The longer, the better. NOTE: When you change the length of this array, change source B in the GRT instruction of rung 1 in GenerateSequentialNumber to match it)

**Lane_Speed_Factor** is unused. I missed it during cleanup ;)

# Documentation/How it works

<h3>Overview</h3>

All logic for this project is in LD and was placed in MainProgram. 

Switching between screens, high scores, and game initialization are all handled in MainRoutine. Whenever the game is running, the (you guessed it!) RunGame routine is executed every cycle. From there (and with the help of the other subroutines) the logic checks for player blocks and collisions, controls player movement (depending on button input), moves the screen down when applicable and at the appropriate speed, and animates the cars moving along the lanes. 

<h3>Rows</h3>
At any time, there are 11 row objects in the HMI (global object) each corresponding to a _Row_ object in the controller (user-defined type) stored the array **Rows**. The structure of the _Row_ UDT is as follows:

Row
  Type: BOOL
  Trees: Tree[13]
  Cars: Car[7]
  Speed: INT
  Car_Offset: INT
  Car_Density: REAL
  Direction: BOOL

All rows are shifted down the HMI screen at **Scroll_Speed**. When **Row_Offset** exceeds 50, it resets to 0 (the rows are jerked back by 50 - the height of one row) and all elements in  **Rows** are shifted down the array. A new row is generated, and its _Type_ is randomized. Depending on the type, other properties of the new _Row_ are also randomized. The moment this happens (Row_Offset=0; JSR ShiftRows) in RunGame, the new _Row_ comes down from the top of the screen. Likewise, the properties of the _Row_ that goes down beyond the screen are lost forever as they are overwritten by those of the row above it. 

To be clear, the position on the HMI screen of the elements of **Rows** in relation to each other remains the same. All are shifted down in unison by **Row_Offset**.

If _Row.Type_ = 0, any of its active trees are visible and can block the player, and the green turf is visible. 

If  _Row.Type_ = 1, any of its active cars are visible,  animated, and can hit the player, and its gray road is visible. Cars are moved horizontally along the screen analogously to the way the Rows are moved vertically. All cars are evenly spaced (the inactive cars are gaps) and move in unison by Car_Offset. If Car_Offset exceeds 110 (ie. the first car is completely on the screen, and the last car is completely off of it) Car_Offset is reset to zero, the Cars are shifted down the array, and the properties of the first car (Active) are randomized. _Car_Density_ is the probability that the next car will be active 

_Row.Direction_ is unused. 
<h3>Cars</h3>
Originally, I wanted to randomize the appearance of cars as they were generated. That is why the _.Cars_ property of **Row** is a UDT and not just BOOL. For reasons I will explain later, I did not do this.  UDT _Car_ has two properties: _Type_ and _Active_. Only _Active_ is used. If a car is active, it can affect the player and is visible on the screen. 

<h3>Trees</h3>
Something similar can be said for the _Tree_ object. It has both _.Type_ and _.Active_, but only _.Active_ is used. Instead of simplifying this, I vied for flexibility and kept it as it was just in case. If a tree is active, it is visible and can block the player's movement. 

<h3>Random Numbers</h3>

At first, I used the current wall clock time in microseconds to generate random numbers. This seemed to result in a lot of repetition, however, so I changed **GenerateSequentialNumber** to flip through an array of random numbers generated elsewhere (**Sequential_Numbers**). When **Sequential_Numbers** was sufficiently large, the patterns became much less pronounced. 

# Room for improvement

A working game was my goal for this project, and for that it is pretty minimal. I think it is very likely that during gameplay ideas I haven't thought of will come to you. But here are a few obvious things: 

- The cars, and occasionally rows, appear to jerk back on the HMI: For the cars, I suspect this is from a delay between the MOV and last COP instruction of rung 2 in AnimateLane, and I think something similar is going on in the row shifting operation. As far as I know, the HMI update is not in sync with the controller cycles, so there is a small chance that the HMI will happen to sample the variables in the middle of a shifting operation. At least for the cars, putting the MOV instruction at the end of the COPs seems to help, but it still happens frequently. Rest assured, a car jerking back or forward should not squash your avatar.

- The characters are not parameterized: If you wanted to change one of the characters (I got mine from symbol factory and the library in factoryTalk view), you would have to do it in three places: Once on the game itself and twice in on the home screen. Not only that, but these are in (nested for the homescreen) groups, have animations, and are set to different sizes. It was too late when I saw that Yoseph didn't have a mouth! Maybe this could be made easier with global objects.
  
- Changing the group structure of the HMI global row object is tedious: In a similar vein, if you ungroup the global row object and change it, not only would have to regroup it and add the parameter name again, but re-add each row to the MAIN display and set each of their parameter values to the corresponding row. I have not found an easy way around this, and it has prevented me from making aesthetic changes.
  
- The cars only move in one direction: Need I say more?
  
- All the cars and trees are identical: Originally this is not what I was going for, but I just wasn't feeling like ungrouping the global row object (see the above) and making things more complicated by turning one tree into three overlayed trees of different types, and likewise for the cars.  On the logic side, all that would need to be done is to randomize tree types and car types.

- 

