---
lesson: 4
layout: default
title: Synaptic Model
---
# A synaptic model

In this lesson, we will be stimulating our neuron with a synaptic input instead of current injection.  There are two types of synaptic input available in NEURON:
1. **AlphaSynapse** : represents a single conductance transient of a particular peak amplitude that starts at a fixed time. It is not meant to connect neurons but to simulate the impact of a synaptic type of stimulus.
1. **ExpSyn and Exp2Syn** : represent event-driven conductance-changing weighted mechanisms (with single or double decay rate constants, respectively).  They will not activate unless triggered by a Network Connection object (NetCon) where the amplitude is controlled by the NetCon weight but the time course and reversal potential is controlled by the ExpSyn or Exp2Syn. (This will be covered in the next lesson)

In NEURON, these are managed as **Point Process** stimuli but unlike the **IClamp** stimulus which is a square wave, synaptic input is modeled with an alpha function which reaches max conductance after an exponential rise.

![alpha-function]

(*Phase Response Curves in Neuroscience - Scientific Figure on ResearchGate. Available from: https://www.researchgate.net/278702575_fig10_Figure-2-2-Alpha-function-current-stimuli-plotted-with-different-time-constants-and-the [accessed 20 Jul, 2017]*)

## Part A: Using CellBuilder

1. Firstly, we will add an **AlphaSynapse** to our *Ball and Stick cell* created in the first lesson.
1. Launch a fresh version of NEURON via `nrngui`
1. Load up the "Ball and Stick cell" from the file *bs_cell.ses*
1. In the **CellBuilder** window, click on the **Continuous Create** button
1. From the Main Menu window, select **Tools** -> **Point Processes** -> **Managers** -> **Point Manager**
1. In the **PointProcessManager** window, click on **SelectPointProcess**, then select **AlphaSynapse**
1. By default, the point process is placed on the soma. To change this click on **Show** then **Shape**
1. Click at the other end of the line which places the synapse at the end of the dendrite. You should see:

```
AlphaSynapse[0]
at: dend[1]
```

![alphasyn]

Click on **Show** then **Parameters** which shows the panel for control of the **AlphaSynapse** which has four parameters:

| Parameter | Description    | Value | Units |
| :------------- | :------------- | :------------- |:------------- |
| `onset` | The time before the change in postsynaptic conductance begins | `0`|ms|
| `tau` | The time to peak of the conductance change | `0.1`| ms|
| `gmax` | The peak conductance change | `10`|µS|
| `e` | The reversal potential for the synaptic current | `-15`|mV|

1. Enter the values as shown in the table
1. Open a **RunControl** and **Graph -> voltage axis**
1. Run with **Init &amp; Run** where `Tstop` = `10`

![simplealpha_params]  ![simplealpha]  

#### Save Me

You can save this to a file called **bs_cell_alpha.ses**.


## Part B: Understanding HOC

We are now ready to take a look under the hood of NEURON at the code which runs the simulator.  The scripting code is written in **HOC (High Order Calculator)** which is an interpretive language loosely resembling C code developed originally as a UNIX interpreter for calculations.  It is now only found in NEURON.  

As we go through the lesson, it will become clear that the GUI version of NEURON (CellBuilder, etc) is actually generating HOC code to run.  As the models you create become more complex and you need to add specific distributions of ion channels or synapses or even if you are just keen to try out some of the many models available online, it is important that you know how to understand the HOC code.

**Recommended structure of a HOC program:**

1. specify model topology (create sections, connect sections)
1. specify model geometry (stylized (L, diam) or pt3d method as appropriate)
1. specify instrumentation (IClamps, SEClamps, other Point Processes, graphs)
1. specify simulation flow control (RunControl panel is sufficient for most purposes)

#### Commands in the console

A console window running the HOC interpreter is launched when `nrngui` is run. HOC code can be run directly in this console which can be useful.  In the console window, check you see `oc>` (if not, just press "Enter" on the keyboard) then type:

```
oc> soma psection()
```

![console_soma]

The output will be explained as we investigate HOC further.


### STEP 1: Generate a cell template

Creating a cell type template, allows us to create one or more of the same cell type which can then be used in network models as well as allowing more complexity to be added.  From our CellBuilder model, we can generate HOC code to get started.

1. From the **CellBuilder** window, select **Management** then **Cell Type**
1. Click on **Classname** and enter `BScell`
1. Click on **Save hoc code in file** and provide a filename (and path) - here we are calling it *bscell.hoc*

We are now going to see what has been generated.

### STEP 2: Understanding the HOC code

1. Open a plain text editor of your choice (a few recommendations are on the [Setup](setup) page)
2. Open the *bscell.hoc* file
1. Find the lines starting `begintemplate` and `endtemplate` at the top and bottom of the file

```
begintemplate BScell
...
endtemplate BScell
```
#### Variables

>If you have not had any exposure to programming constructs, have a read of the [Help page] with the aim that you will be able to understand the code rather than having to write it.

+ **Global variables** are indicated by the starting word: `public`. These are available anywhere in the code. You should recognize a few names from our CellBuilder model: `soma`, `dend`
+ **Local variables** are indicated by the starting word: `objref` or `objvar`. These are only available in this file.

```c
public soma, dend
public all
```
#### Program Sequence

When the code is first loaded in the NEURON interpreter, it runs **procedures** which are sets of instructions.

1. Find where the code first creates the `soma` and `dend` sections (they are empty at this point):

```c
create soma, dend
```

1. Now find the procedure (shown as `proc`) called `init()` (which stands for initialize). By default, this is the first procedure run.
>Note that comments can be added with "//" in front

```c
proc init() {
  topol()               //Step 1
  subsets()             //Step 2
  geom()                //Step 3
  biophys()             //Step 4
  geom_nseg()           //Step 5
  synlist = new List()  //Step 6
  synapses()            //Step 7
  x = y = z = 0 // only change via position
}
```

Each procedure listed inside the `{}` in `init()` is run in sequential order.  
So the first call is to `topol()` (topology).
1. Look in the code to see what `topol()` is defined as:

```c
proc topol() { local i
  connect dend(0), soma(1)   //Connects dendrite to soma
  basic_shape()              //Calls another procedure
}
```

So here we see that the `dend` is connected to the `soma` object.
>This was where we added the soma and dendrite in the CellBuilder under the Topology tab.

Each section is indexed from 0 to 1 along it's length so that:
+ `dend(0)` refers to the **start of the section** and
+ `dend(1)` refers to the **end of the section**.  

![connect]

Within the procedure, there is a call to another procedure: `basic_shape()` so it is fine to embed procedures within each other.  The remaining procedures are all run in sequence until the end is reached.

### STEP 3: Adding an axon section via HOC

Now if we want to add an **axon**, we can add this to the code rather than returning to **CellBuilder**.

>OPTIONAL: Save a copy to **bscell_orig.hoc** as we will be making changes to this file.

| Section | Ref | Length (&micro;m) | Diameter (&micro;m) | Biophysics |
| ---- | ---- | ---- | ---- | ----|
| Axon | `axon` | 800 | 1 | hh |

1. Declare `axon` as a variable - append this to the line `public soma, dend`

```c
public soma, dend, axon
```

1. Create the axon section

```c
create soma, dend, axon
```

#### Topology

1. Add `axon` in the `topol()` and `basic_shape()` procedures

```c
proc topol() { local i
  connect dend(0), soma(1)
  connect axon(0), soma(0)     //Add this line
  basic_shape()
}

proc basic_shape() {
  //format is pt3dadd(x,y,z,diam)
  soma {pt3dclear() pt3dadd(0, 0, 0, 1) pt3dadd(20, 0, 0, 1)}
  dend {pt3dclear() pt3dadd(20, 0, 0, 1) pt3dadd(120, 0, 0, 1)}
  //Add this line
  axon {pt3dclear() pt3dadd(0, 0, 0, 1) pt3dadd(-120, 0, 0, 1)}    
}

```
#### Geometry

<button class="btn btn-info" data-toggle="collapse" data-target="#tipG">Geometry parameters ...</button>
<div id="tipG" class="alert alert-success collapse">
<p>It is important to refer to the <code class="highlighter-rouge">soma</code> and <code class="highlighter-rouge">dendrite</code> as <em>sections</em> rather than <em>segments</em>. A <strong>section</strong> contains one or more segments (<code class="highlighter-rouge">nseg</code>) and it is by dividing the section this way into smaller compartmental units which can provide the fine-grained detail underlying a more realistic model.</p>

<p>Each section has the following parameters:</p>

<table>
  <thead>
    <tr>
      <th style="text-align: left">Parameter</th>
      <th style="text-align: left">Symbol</th>
      <th style="text-align: left">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align: left">Length</td>
      <td style="text-align: left">L</td>
      <td style="text-align: left">For each section, L is the length of the entire section in microns.</td>
    </tr>
    <tr>
      <td style="text-align: left">Segments</td>
      <td style="text-align: left">nseg</td>
      <td style="text-align: left">The section is divided into nseg compartments of length L/nseg. Membrane potential will be computed at the ends of the section and the middle of each compartment.</td>
    </tr>
    <tr>
      <td style="text-align: left">Diameter</td>
      <td style="text-align: left">diam</td>
      <td style="text-align: left">The diameter in microns. Note that diam is a range variable and therefore must be respecified whenever nseg is changed.</td>
    </tr>
    <tr>
      <td style="text-align: left">Resistivity</td>
      <td style="text-align: left">Ra</td>
      <td style="text-align: left">Axial resistivity in ohm-cm.</td>
    </tr>
    <tr>
      <td style="text-align: left">connectivity</td>
      <td style="text-align: left">connect</td>
      <td style="text-align: left">Defines the parent of the section, which end of the section is attached to the parent, and where on the parent the attachment takes place .</td>
    </tr>
  </tbody>
</table>
</div>


1. Specify the length and diam in the `geom()` procedure.

```c
proc geom() {
  forsec all {  }
   soma.L = 20
   dend.L = 1000
   soma.diam = 20
   dend.diam = 5
   axon.L = 800             //Add this
   axon.diam = 1            //Add this
}
```

>This was where we specified the Geometry in the CellBuilder.

#### Biophysics

1. To specify *hh* in the `biophys()` procedure, make a copy of the `soma` code and call it `axon`.


```c
proc biophys() {
  forsec all {
     Ra = 160
     cm = 1
   }
   soma {
     insert hh
       gnabar_hh = 0.12
       gkbar_hh = 0.036
       gl_hh = 0.0003
       el_hh = -54.3
   }
   dend {
     insert hh
       gnabar_hh = 0.012
       gkbar_hh = 0.0036
       gl_hh = 0.0003
       el_hh = -64
   }
   axon {                      //Copy soma here and rename as axon
     insert hh                 
       gnabar_hh = 0.12        
       gkbar_hh = 0.036
       gl_hh = 0.0003
       el_hh = -54.3
   }
 }

```

#### Repeated actions with forsec

HOC provides a **for loop** type construct to apply repeated actions to multiple sections. Most commonly this is the `forsec` (for section) function for example, `forsec all` which can also be written as `forall`.

>This was where we specified the Biophysics in the CellBuilder.

#### SectionList

Sections are added to a list called a **SectionList** to group components together under one reference name.
1. In the `subsets()` procedure add the `axon` to the SectionList called `all`

```c
proc subsets() { local i
  objref all
  all = new SectionList()
    soma all.append()
    dend all.append()
    axon all.append()
}
```


#### Different possible notations which all do the same thing:

| Notation | Example |Explanation     |
| :------------- | :------------- |:------------- |
| Space Notation | `axon` `diam`  = 20     | separated by space       |
| Dot Notation | `axon.diam` = 20    | separated by dot |
| Stack Notation | `axon` { `diam` = 20 } | grouped with other parameters by {} |
| Default access | `access axon` |  sets the default section ... |
| |  `diam = 20` | ...so it can be followed without further specifying |


------------------

#### Save Me

You can save your changes to **bscell.hoc**


> Hopefully, it is now apparent that the HOC code is underlying the tasks undertaken in CellBuilder.

### STEP 4: Adding an AlphaSynapse via HOC

1. Locate the procedure `synapses()` near the end of the file. We will enter our code within the `{}`

```c
proc synapses() {}
```

To insert an AlphaSynapse, we create an instance of it with

```c
objectvar syn
syn = new AlphaSynapse(x)
```
where `x` is the location on the section.

So to insert an AlphaSynapse at the distal end of the dendrite, we specify the section first and the location as `1`:

```c
objectvar syn
dend syn = new AlphaSynapse(1)
```

The properties are set for the `syn` object as per our table, similar to the parameters window in the previous lesson:

```c
syn.onset = 0     // time to onset in ms
syn.tau	  = 0.1   // rise time constant in ms
syn.gmax  = 10    // peak conductance	in &mmicro;S (umho)
syn.e	    = -15   // reversal potential in	mV
syn.i     = 0     //	nA
```

1. So now go ahead and enter the final code as follows:

```c
objectvar syn       // this must be outside the procedure
proc synapses() {
  // add synapse to the dendrite
  dend syn = new AlphaSynapse(1) // position is end of dendrite
  syn.onset = 0     // time to onset in ms
  syn.tau	  = 0.1   // rise time constant in ms
  syn.gmax  = 10    // peak conductance	in %mmicro;S (umho)
  syn.e	    = -15   // reversal potential in	mV
  syn.i     = 0     //	nA
}

```

> This should look familiar to the AlphaSynapse PointProcess window in the beginning of the lesson.

#### Save Me

You can save your changes to **bscell.hoc**

### STEP 5: Running the HOC code

So now how do we get this code into NEURON?   

To recap, what we have done so far is create a *template* for a neuron.  The neuron itself will not exist until we run a command to create an object from the template. We need to create a script which loads all the required files and runs the initial code. This will be loaded into NEURON.

```c
objectvar bs

bs = new BScell()      //create an object bs from template BScell
bs init()              //not strictly necessary but ensures it is run
```

1. Create a file called `bs_run.hoc` which we will use to set everything up.
1. We will need to load the NEURON libraries via `nrngui.hoc`
1. Then we load our template file `bscell.hoc`
1. Then we add our code above to create an instance of our template called `bs`

The contents of `bs_run.hoc` should look like:

```c
/* bs_run.hoc for BScell
*/

// start the GUI
load_file("nrngui.hoc")
// load our hoc file
load_file("bscell.hoc")
//=================================================
// Create our cell from the template
//=================================================
objectvar bs

bs = new BScell()
bs init()

//==================================================
//Run Control
//==================================================
tstop = 20
dt = 0.025 //ms 40khz

```

(*Note we have also set a couple of variables for the RunControl panel*)

1. Now relaunch NEURON (`nrngui`)
1. Reset working dir with **File -> recent dir**
1. Open `bs_run.hoc` with **File -> load hoc**
1. All you should now see is a console window as shown here (any errors will show up here):

![console_init]

To test that our code has actually loaded we will use the console commands:

```c
oc> bs
oc> bs.soma psection()
oc> bs.dend psection()

```

This should show the following output:

![console_bsa]

If it all goes well, you can see that
+ the `bs` object is our template `BScell`
+ the `soma` has all the properties we set in the template code
+ the `dend` has the properties we set and also has 21 `nseg` (segments) which has been determined by the `d_lambda` option.

Alternatively, to see all sections:
```c
oc> forall {psection()}
```

#### Viewing the model

Unfortunately, we can no longer use the CellBuilder as although there is an *import* function, it is fairly limited.  So how do we see what we have built?

1. From the Main menu, select **Tools -> Model View**
1. Click on the line **1 real cells** then **BScell**
1. Further lines will continue to reveal more information.
1. Our AlphaSynapse can be seen in both the **BScell** region and under **Density Mechanisms**

![modelview]


#### Runnning the Simulation

As previously, we will use the Run Control and Graphical windows to view our simulation.

1. Open the **File -> RunControl** (you will see our parameters in the `bs_run.hoc` are set)
1. Open a **Graph -> Voltage Axis** window
1. Click on **Init &amp; Run** -> Voila!

![hocoutput]

> CONGRATULATIONS! This is no mean feat and you deserve a MEDAL!!


------------------------
### Experiments

1. How would you generate more than one BScell for a network?

1. Try changing the properties of the AlphaSynapse to see the effect on the AP, for example, introduce a delay to the onset to allow for equilibration. How could these properties be changed dynamically?
(*Note the template hoc file must be reloaded with each change by restarting NEURON. Alternatively, if the AlphaSynapse code is moved to bs_run.hoc, it can be reloaded with `xopen("bs_run.hoc")`*)

1. How would you add multiple dendrites?

1. How might you make the synapse generation more dynamic?

```
//==================================================
//Synapses proc with arguments
//==================================================
objectvar syn       
proc synapses() {
  dend syn = new AlphaSynapse($1) // where $1 is an argument
  syn.onset = 100     // time to onset in ms
  syn.tau	  = 0.1   // rise time constant in ms
  syn.gmax  = 10    // peak conductance	in %mmicro;S (umho)
  syn.e	    = -15   // reversal potential in	mV
  syn.i     = 0     //	nA
}

//The function could then be called with
synapses(1)
//or maybe if the position was changed to 0.4
synapses(0.4)

```

--------
## Resources

[Point Process](https://www.neuron.yale.edu/neuron/static/docs/help/neuron/neuron/mech.html#pointprocesses)

[HOC keywords](http://www.neuron.yale.edu/neuron/static/new_doc/programming/ockeywor.html)

[HOC syntax](http://www.neuron.yale.edu/neuron/static/new_doc/programming/hocsyntax.html)

[Programming HOC guide](http://www.neuron.yale.edu/neuron/static/new_doc/modelspec/programmatic.html)

[AlphaSynapse](https://www.neuron.yale.edu/neuron/static/docs/help/neuron/neuron/mech.html#AlphaSynapse)

[ExpSyn](https://www.neuron.yale.edu/neuron/static/docs/help/neuron/neuron/mech.html#ExpSyn)

[Exp2Syn](https://www.neuron.yale.edu/neuron/static/docs/help/neuron/neuron/mech.html#Exp2Syn)

[Geometry](https://www.neuron.yale.edu/neuron/static/docs/help/neuron/neuron/geometry.html#Geometry)

[Section](https://www.neuron.yale.edu/neuron/static/docs/help/neuron/neuron/secspec.html)

[SectionList](https://www.neuron.yale.edu/neuron/static/docs/help/neuron/neuron/classes/seclist.html#SectionList)

[Help page]({{ site.github.repository_url }}/help)

[alpha-function]: {{ site.github.repository_url }}/raw/gh-pages/img/Alpha-function.png "Alpha function"

[alphasyn]: {{ site.github.repository_url }}/raw/gh-pages/img/Alphasynapse_shape.PNG "AlphaSynapse shape"

[simplealpha]: {{ site.github.repository_url }}/raw/gh-pages/img/Simplealpha.PNG "Simple AlphaSynapse"

[simplealpha_params]: {{ site.github.repository_url }}/raw/gh-pages/img/Simplealpha_params.PNG "Simple AlphaSynapse Parameters"

[axoncell]: {{ site.github.repository_url }}/raw/gh-pages/img/Axoncell.PNG "Axon added to BS cell"

[console_soma]:{{ site.github.repository_url }}/raw/gh-pages/img/Console_soma.PNG "Console with soma"

[connect]:{{ site.github.repository_url }}/raw/gh-pages/img/sthAeqsec.gif "Connecting sections"

[console_init]:{{ site.github.repository_url }}/raw/gh-pages/img/Console_init.PNG "Console init"

[console_bsa]:{{ site.github.repository_url }}/raw/gh-pages/img/Console_bsa.PNG "Console with bsa output"

[hocoutput]:{{ site.github.repository_url }}/raw/gh-pages/img/hocoutput.PNG "Running our HOC code"

[modelview]:{{ site.github.repository_url }}/raw/gh-pages/img/ModelView.PNG "Model View"
