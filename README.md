# Crowds Wrangle Scripts
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
