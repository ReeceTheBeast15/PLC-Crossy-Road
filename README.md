# PLC-Crossy-Road
Play a game based on Crossy Road with Studio5000 and FactoryTalk View!

<h2>How to play</h2>
When testing and building this project, commmunications were set up as follows: 
<br><br>
In RSLinx Classic Lite --> configure drivers, "Virtual Backplane (SoftLogix58xx USB)" was selected and added with the default name in slot 1. <br>
In Logix Emulate,  Emulate 5570 (V30), was added to slot 0 of the virtual backplane. After this the Logix Emulate window was as shown in Figure1.png. 
<br><br>
By default, the communications in Crossy_Road.mer and Crossy_Road.api are configured accordingly, but of course you may change them for whatever controller you want.
<br><br>
Download and extract CrossyCode.zip. Open Crossy_Road.ACD; download program to the controller and put in run mode. Open and run Crossy_Road.mer. 
When you select a character and hit 'PLAY', control player movement with the four buttons on the lower right-hand corner. 

<h2>Setting parameters</h2>
You may want to adjust the game to fit your preferences. Crossy_Road.ACD has a few controller tags that make this easy.
<br><br>

**Scroll_Speed** sets the speed that the rows are shifted down the screen. 

**Lane_Density** sets the percentage of rows that are roads/lanes (_Type_=1). **Lane_Density** = 1 is only lanes, and **Lane_Density** = 0  is no lanes.    

**Lane_Speed_Min** and **Lane_Speed_Max** set the minimum and maximum value of _Speed_ in new lanes. When the first row is randomized and it happens that _Type=0_, its speed (in theory) is selected randomly from a uniform distribution between these values. 

**Lane_Car_Density_Min** and **Lane_Car_Density_Max** are analogous to **Lane_Speed_Min** and **Lane_Speed_Max**, but for the _Car_Density_ property. 

**Tree_Density** determines the proportion of trees that will be active in newly generated rows where _Type_=0. **Tree_Density** = 1 is all trees, and **Tree_Density** = 0 is no trees. 

**Collision_Offset_Min** and **Collision_Offset_Max** are used to determine how easily a car hits the player. **Collision_Offset_Min** is the minimum value of _Car_Offset_ required for an active car in the car column to the left of the player in the same row to hit the player. Setting it lower will extend the range the car hits the player in front of it. Likewise, **Collision_Offset_max** is the maximum _Car_Offset_ required for an active car in the car column to the right of the player in the same row to hit the player. Setting it higher will extend the range the car hits the player behind it. 

**Grass_Strip** is the initialization of the bottom three rows at the beginning of the game (see MainRoutine). 

**Random_Mode** determines how random numbers are obtained. When equal to 0 (default) they are generated sequentially from a list (GenerateSequentialNumber). When equal to 1, they come from the wall clock time (GenerateRandomNumber). 

**Random_Number_Divisor** only applies when **Random_Mode** = 1. Increasing its value will result in more possible values returned from GenerateRandomNumber. 

**Sequential_Numbers** only applies when **Random_Mode** = 0. It is a list of the random numbers flipped through by GenerateSequentialNumber. If you make it longer you will see less patterns. NOTE: When you change the length of this array, change source B in the GRT instruction of rung 1 in GenerateSequentialNumber to the new length).

**Lane_Speed_Factor** is unused.

<h2> Documentation/How it works</h2>

<h3>Overview</h3>

All game logic is contained in the LD routines of MainProgram. 

Screen switching, high scores, and game initialization are all handled in MainRoutine. The RunGame routine is executed from there every cycle that the game is running. From RunGame (and with the help of the other subroutines) the logic checks for player blocks and collisions, controls player movement (depending on button input), moves the screen down, and animates the cars moving along the lanes. 

<h3>Rows</h3>

At any time, there are 11 instances of the global object row in the HMI, each corresponding to a UDT Row in the controller stored the array **Rows**. The structure of the Row UDT is as follows:


- Type: BOOL
- Trees: Tree[13]
- Cars: Car[7]
- Speed: INT
- Car_Offset: INT
- Car_Density: REAL
- Direction: BOOL

All rows are shifted down the HMI screen by **Row_Offset** at **Scroll_Speed**. When **Row_Offset** is greater than 50 (the height of one row), it is reset to 0 and all elements in  **Rows** are shifted down the array. A new row is generated, and its _Type_ is randomized. Depending on the type, other properties of the new row are also randomized. On the HMI screen, the new row comes down from the top of the screen. Likewise, the properties of the _Row_ that goes down beyond the screen are lost forever as they are overwritten by those of the row above it. 

If _Row.Type_ = 0, any of its active trees are visible and can block the player.  

If  _Row.Type_ = 1, any of its active cars are visible and animated. Cars are moved horizontally along the screen at _Speed_ analogously to the way the rows are moved vertically. All cars are evenly spaced (apparent gaps are made by inactive cars) and move in unison by Car_Offset. If Car_Offset exceeds 110 (ie. the first car is completely on the screen, and the last car is completely off of it) **Car_Offset** is reset to zero, the cars are shifted down the array, and the _Active_ property of the first car is randomized according to _Car_Density_. 

_Row.Direction_ is unused. 
<h3>Cars</h3>

UDT _Car_ has two properties: _Type_ (INT) and _Active_ (BOOL). Only _Active_ is used. If a car is active, it can hit the player and is visible on the screen. 

<h3>Trees</h3>

UDT _Tree_ has two properties: _Type_ (INT) and _Active_ (BOOL). Only _Active_ is used. If a tree is active, it can block the player and is visible on the screen. 

<h3>Random Numbers</h3>

At first, I used the current wall clock time in microseconds to generate random numbers. This seemed to result in a lot of repetition, however, so I changed **GenerateSequentialNumber** to flip through an array of random numbers generated elsewhere (**Sequential_Numbers**). When **Sequential_Numbers** was sufficiently large, the patterns became much less pronounced. 

<h2>Room for improvement</h2>

A working game was my goal for this project, and so it is quite minimal. I think it is very likely that during gameplay ideas I haven't thought of will come to you. But here is what sticks out to me: 

- The cars, and occasionally rows, appear to jerk back on the HMI: For the cars, I suspect this is from a delay between the MOV and last COP instruction of rung 2 in AnimateLane, and I think something similar is going on in the row shifting operation. As far as I know, the HMI update is not in sync with the controller cycles, so there is a small chance that the HMI will happen to sample the variables in the middle of a shifting operation. At least for the cars, putting the MOV instruction at the end of the COPs seems to help, but it still happens frequently. Rest assured, a car jerking back or forward should not squash your avatar.

- You have to press buttons with a mouse to move: I searched for and could not find a way to change tags with keypresses in View ME. In view SE, on the other hand, it could easily be done with macros. 

- The characters are not parameterized: If you wanted to change one of the avatars, you would have to do it once on the game screen  and twice in on the home screen. In each case you would have to ungroup, regroup, and add animations. It was too late when I saw that Yoseph didn't have a mouth. 
  
- Changing the group structure of the HMI global row object is tedious: In a similar vein, if you ungroup the global row object and change it, not only would have to regroup it and add the parameter name again, but re-add each row to the MAIN display and set each of their parameter values to the corresponding row. I have not found an easy way around this, and it has prevented me from making aesthetic changes.

- The cars only move in one direction: Need I say more?
  
- All the cars and trees are identical: Originally this is not what I was going for, but I just wasn't feeling like ungrouping the global row object (see the above) and making things more complicated by turning one tree into three overlayed trees of different types and doing the same for the cars.  On the logic side, all that would need to be done is to randomize tree types and car types.
