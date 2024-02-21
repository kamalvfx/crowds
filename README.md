# Crowds VEX Scripts
*A collection of wrangle scripts and python tools for crowds.*
<p align="right"><small>by Kamaljeet Singh</small></p>

* [collect all agent names](#collect-all-agent-names)
* [switch variant with another variant](#switch-variant-with-another-variant)
* [switch variant with another variant within family](#switch-variant-with-another-variant-within-family)
* [switch clip or set another clip](#switch-clip-or-set-another-clip)
* [switch clip if clip is missing](#switch-clip-if-clip-is-missing)
* [set clip offset according to current clip duration](#set-clip-offset-according-to-current-clip-duration)
* [Assign Layers by Row in a Grid](#assign-layers-by-row-in-a-grid)
## collect all agent names
```c
string agentnames[] = uniquevals(1, "point", "agentname");
s[]@names = agentnames;
```
## switch variant with another variant
```c
// IN 1: template points
// IN 2: agents

string agentnames[] = uniquevals(1, "point", "agentname");
s[]@names = agentnames;

int choice = int(rand(@ptnum + chf("shuffle_seed")) * len(s[]@names));
s@agentname = s[]@names[choice];

```
## switch variant with another variant within family

```c
int choice = int(rand(@ptnum * chf("shuffle_seed")) * len(s[]@names));
string familyname = split(s@agentname, "_")[0];
string pattern = familyname + "*";
string variants[];
foreach(string name; s[]@names) {
    if (match(pattern, name)) {
        append(variants, name);
        }
    }
if (match(pattern, s@agentname)) {
    choice = int(rand(@ptnum * chf("shuffle_seed")) * len(variants));
    s@agentname = variants[choice];
    }
```
## switch clip or set another clip
```c

// IN 1: template points
if(chi("useclipname")) {
    string targetclip = chs("targetclipname");
    setagentclipnames(0, @primnum, array(targetclip));
    s@state = targetclip;
    }
else {
    string clipnames[] = agentclipcatalog(0, @primnum)[1:];
    int choice = int(rand(@primnum + chf("shuffle_seed")) * len(clipnames));
    string targetclip = clipnames[choice];
    setagentclipnames(0, @primnum, array(targetclip));
    s@state = targetclip;
    }

```

## switch clip if clip is missing
```c
// IN 1: template points

string currentclip = agentclipnames(0, @primnum)[0];
if (currentclip == "Take_001") {
    if(chi("useclipname")) {
        string targetclip = chs("targetclipname");
        setagentclipnames(0, @primnum, array(targetclip));
        s@state = targetclip;
        }
    else {
        string clipnames[] = agentclipcatalog(0, @primnum)[1:];
        int choice = int(rand(@primnum + chf("shuffle_seed")) * len(clipnames));
        string targetclip = clipnames[choice];
        setagentclipnames(0, @primnum, array(targetclip));
        s@state = targetclip;
        }
    }
```

## set clip offset according to current clip duration
```c
string clip = agentclipnames(0, @primnum)[0];
float length = agentcliplength(0, @primnum, clip);
float starttime = chf("start_frame");
float cliptimes[];
cliptimes[0] = fit01(rand(@primnum + chf("seed")), 0, length) + @Time;
setagentcliptimes(0, @primnum, cliptimes);
```

## Assign Layers by Row in a Grid
```c
int factor = int((@ptnum/chi("agents_in_row"))) % chi("num_of_props");
string layers[] = agentlayers(0, @primnum);
setagentcurrentlayers(0, @primnum, array(layers[factor]));
```

## switch current clip with mirror clip
```c
string currentclip = agentclipnames(0, @primnum)[0];
if (match(chs("source_clip"), currentclip)) {
    //s[]@clipname = array(currentclip + "mirror");
    string clipname[] = split(currentclip, "_");
    string mirrorclip = clipname[0] + "_" + clipname[1] + "mirror_" + clipname[2];
    setagentclipnames(0, @primnum, array(mirrorclip));
    //setpointattrib(0, "P", @ptnum, chv("position_offset"), "set");
    setpointgroup(0, "lumCallumMiror", @ptnum, 1);
    }
```

## Add time offset to current cliptimes
```c
float currenttime = agentcliptimes(0, @primnum)[0];
float offset = fit01(rand(@ptnum * chf("seed")), 0, chf("max_offset"));
float settime[] = array(currenttime + offset);
setagentcliptimes(0, @primnum, settime);
```

## Set heading using target geometry
```c
vector pos1 = point(0, "P", @ptnum);
vector pos2 = point(1, "P", 0);
vector dir = normalize(pos2 - pos1);
dir[1] = 0;
v@heading = dir;
@N = v@heading;        // to visualize orientation
```

## Filter Agents by Proximity
```c
// point filter tool 1.0
// authored by Kamaljeet Singh
// set Run Over parm to "Detail (only once)"
int pair_points[];
for (int i = 0; i < npoints(0); i++) {
    if (find(pair_points, i) > -1) {
        continue;
        }
    vector pos = point(0, "P", i);
    string agentname = point(0, "agentname", i);
    int near_points[] = nearpoints(0, pos, chf("max_distance"));
    pop(near_points, 0); // get rid of self point number
    foreach (int near_point; near_points) { 
        string near_agentname = point(0, "agentname", near_point);
        if (agentname == near_agentname) {
            append(pair_points, near_point);
            }
        }
    }    
foreach (int cull_point; pair_points) {
    setpointattrib(0, "Cd", cull_point, {1, 1, 0}, "set");
    setpointgroup(0, "filter", cull_point, 1, "set");
    }
```

# Crowd Shelf Tools
## AutoFill Agent Names
```python
# add agent names on crowd source node with custom distribution
import hou

node = hou.selectedNodes()[0]
geo = node.geometry()

input_node = node.inputs()[0]
input_node_geo = input_node.geometry()
names = input_node_geo.pointStringAttribValues("agentname")
count = len(names)

node.parm("randomizeagent").set(1)
node.parm("randomizeagentmethod").set(0)
node.parm("numagentpatterns").set(count)

for i in range(count):
    node.setParms({"agentnames_%d" % (i+1): names[i]})
```

## AutoFill Clip Names
```python
# add agent clip names on crowd source node with custom distribution

import hou

node = hou.selectedNodes()[0]
geo = node.geometry()

input_node = node.inputs()[0]
input_node_geo = input_node.geometry()

clipnames = []
points = input_node_geo.points()
for point in points:
    prim = point.prims()[0]
    primclipnames = prim.intrinsicValue("agentclipcatalog")
    for primclipname in primclipnames:
        if primclipname not in clipnames:
            clipnames.append(primclipname)
 
# skip rest clip           
clipnames = clipnames[1:]
count = len(clipnames)

node.parm("randomizedefaultstate").set(1)
node.parm("randomizestatemethod").set(0)
node.parm("numstatepatterns").set(count)

for i in range(count):
    node.setParms({"statepattern_%d" % (i+1): clipnames[i]})
```

## AutoCreate State DOPs
```python
import hou
try:
    node = hou.selectedNodes()[0]
    geo = node.geometry()
    
    input_node = node.inputs()[0]
    input_node_geo = input_node.geometry()
    stateattribvals = input_node_geo.pointStringAttribValues("state")
    
    states = []
    for val in stateattribvals:
        if val not in states:
            states.append(val)
            
    states.sort()
    
    for state in states:
        statenode = node.createNode("crowdstate::3.0")
        statenode.setName(state, unique_name="True")
        statenode.parm("cliptype").set(1)
        statenode.moveToGoodPosition(move_unconnected=False)

except IndexError:
    hou.ui.displayMessage("Select DOP Network Node!")
```
## Clear Anim for All But First (Agent Manager)
```python
# clear anim for all variants except first one
node = hou.selectedNodes()[0]

count = node.parm("num_agents")
count = count.eval()

for i in range(2, count+1):
    parmname = "clear_anim_files"+str(i)
    parm = node.parm(parmname)
    parm.pressButton()
```
## Add Anim For All (Agent Manager)
```python
# button clicker tool
# update first agent with all clips and run the tool.
# It will add the clips for rest of the agents.

node = hou.selectedNodes()[0]

count = node.parm("num_anim_files1")
count = count.eval()
for i in range(count):
    parmname = "add_cur_anim_to_all_agents1_"+str(i+1)
    parm = node.parm(parmname)
    parm.pressButton()

```
## Add Locomotion Joint
```python
# sometimes agents have no locomotion node name in agent manager.
# dive inside agent manager, select all agent clip nodes and run the tool.

nodes = hou.selectedNodes()

for node in nodes:
    node.setParms({"createlocomotionjoint": 1, "locomotionnode": "crw_locomotion"})
```
# Node Preset
## Agent Edit - Clip Offset
```
ch("offset")+(($FF-101)/$FPS)
```
