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
odgi extract --threads=48 -i chr5.full.og --paths-to-extract=grch38-path-name-chr5.txt -r GRCh38#chr5 -o chr5-grch38.og
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

It also gives the following error (in addition ot the correct answer):

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

**So, node 78332 is the first node AFTER node 77874 in the GRCh38 reference*.**





  
