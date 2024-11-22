## Locating novel chromosome 5 regions in the GRCh38 reference genome

Once novel regions have been identified, it is useful to know where these are relative to anexisting reference genome (i.e., which nodes in the reference genome are adjacent to the novel nodes?)

Extract path name for GRCh38 (or chromosome 5)

```
odgi paths -i chr5.full.og -L | head -2 | tail -1 > grch38-path-name-chr5.txt
more grch38-path-name-chr5.txt
```

```
GRCh38#chr5
```

Extract the nodes for just the GRCh38 path (saves to `chr5-grch38.og`):

```
odgi extract --threads=24 -i chr5.full.og --paths-to-extract=grch38-path-name-chr5.txt -r GRCh38#chr5 -o chr5-grch38.og
```

Get the node IDs that are present in the GRCh38 path

```
odgi view -i chr5-grch38.og -g | grep ^S | cut -d$'\t' -f2 > chr5-grch38-node-list.txt
head chr5-grch38-node-list.txt
```

```
1
2
3441
3442
3444
3445
3446
3447
3448
3453
```

Where is a particular non reference sequence (NRS) node sitiated relative to nodes in teh reference genome?

Let's use the first node we found previously: 77874

What is the highest node ID BEFORE the specified NRS node (77874)?

```
cat chr5-grch38-node-list.txt | awk '{if($1 < 77874) print $1}' | tail -1
```

```
72603
```

What is the node AFTER the NRS node? (node ID is 77874)

**NB** - `exit` will terminate awk after the condition is satisfied for the first time

```
cat chr5-grch38-node-list.txt | awk '{if($1 > 77874) {print $1; exit}}'
```

```
78332
```

It also gives the following error (in addition to the correct answer):

```
cat: write error: Broken pipe
```

It's to do with `awk` closing the pipe while adat is still coming through (probably because of `exit` is being used).

The following works, without the error:

```
awk '{if($1 > 77874) {print $1; exit}}' chr5-grch38-node-list.txt
```

```
78332
```

**So, node 78332 is the first node AFTER node 77874 in the GRCh38 reference.**

Get the GRCh38 position info for the node before (pre-node) and after (post-node) the NRS node of interest.

Pre-node:

```
odgi extract --threads=24 -i chr5.full.og -n 72603 -o chr5-full-node72603.og
```

Post-node:

```
odgi extract --threads=24 -i chr5.full.og -n 78332 -o chr5-full-node78332.og
```

Get GRCh38 coordnates for the pre-node:

```
odgi paths -i chr5-full-node72603.og -L | grep GRCh38
```

```
GRCh38#chr5:778710-778758
```

Get GRCh38 coordnates for the post-node:

```
odgi paths -i chr5-full-node78332.og -L | grep GRCh38
```

```
GRCh38#chr5:778758-778766
```

We can combine the two steps above (i.e., don't save node graph).

Pre-node:

```
NODE=72603
odgi extract --threads=24 -i chr5.full.og -n $NODE -o - | odgi paths -i - -L | grep GRCh38
```

```
GRCh38#chr5:778710-778758
```

Post-node:

```
NODE=78332
odgi extract --threads=24 -i chr5.full.og -n $NODE -o - | odgi paths -i - -L | grep GRCh38
```

```
GRCh38#chr5:778758-778766
```

So the NRS node (in this case) is inserted at position 778758

What is the sequence for these nodes?

```
odgi view -i chr5-grch38.og -g | grep  -w 72603 | grep ^S
```

```
S       72603   AGATCATCTTGGATTATCCACAGCTGAGCCCTAAATCCAATGGTGAGT
```

```
odgi view -i chr5-grch38.og -g | grep  -w 78332 | grep ^S
```

```
S       78332   GTCTCTAC
```

Interesting - node 78332 has coordinates `chr5:778758-778766` which is 9 bases, but there are 
only 8 bases in the sequence above. Let's check:

```
odgi view -i chr5-grch38.og -g | grep  -w 78332 | grep ^S | cut -d$'\t' -f 3 | awk '{print length($0)}'
```

```
8
```

**NB:** have to be careful with `wc -c` for this, as it counts the end of line character:

```
odgi view -i chr5-grch38.og -g | grep  -w 78332 | grep ^S | cut -d$'\t' -f 3 | wc -c
```

```
9
```

So, there are 9bp in chr5:778758-778766, but only 8bp in the node. What is going on? Is the first base (G) at position 778758 or 778759?

Let's look at the first node:

```
NODE=1
odgi extract --threads=24 -i chr5.full.og -n $NODE -o - | odgi paths -i - -L | grep GRCh38
```

```
GRCh38#chr5:0-2252
```

Sequence for node 1:

```
odgi view -i chr5-grch38.og -g | grep  -w 1 | grep ^S
```

```
S       1       NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
```

Counts the N's:

```
odgi view -i chr5-grch38.og -g | grep  -w 1 | grep ^S | cut -d$'\t' -f 3 | awk '{print length($0)}'
```

```
2252
```

So, that implies that the LAST base position is NOT part of the node sequence.

For node 78332 above (chr5:778758-778766, GTCTCTAC), the G is at 778758, and the final C is at 778765.

If we are thinking about NRS nodes, then the final base of the reference sequence is actually one base LESS than the end coordinate for the pre-node. 

For our NRS node (77874) that is preceeded by node 72603 (which has GRCh38 coordinates chr5:778710-778758), node 72603 comes immediately after 778757 (not 778758, which is the first base of node 78332).


### Aside: GFA format

The GFA (-g) format is interesting, because it contains LOTS of useful info 

```
odgi view -i chr5-grch38.og -g | grep  -w 72603 | more
```

```
L       72601   +       72603   +       0M
S       72603   AGATCATCTTGGATTATCCACAGCTGAGCCCTAAATCCAATGGTGAGT
L       72603   +       78332   +       0M
P       GRCh38#chr5:0-181538259 1+,2+,3441+,3442+,3444+,.....,72603+,78332+,....
```

Links:
 
- https://github.com/GFA-spec/GFA-spec
- http://lh3.github.io/2014/07/19/a-proposal-of-the-grapical-fragment-assembly-format

**NB - odgi is using GFAv1**

https://github.com/GFA-spec/GFA-spec/blob/master/GFA1.md

```
# 	Comment
H 	Header
S 	Segment
L 	Link
J 	Jump (since v1.2)
C 	Containment
P 	Path
W 	Walk (since v1.1)
```

Grab node 72603 details, and the next line (which happens to be 78332):

```
odgi view -i chr5-grch38.og -g | grep  ^S | cut -d$'\t' -f2-3 | grep -A 1 -w ^72603
```

```
72603   AGATCATCTTGGATTATCCACAGCTGAGCCCTAAATCCAATGGTGAGT
78332   GTCTCTAC
```


