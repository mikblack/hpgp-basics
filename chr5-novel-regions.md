# Novel regions - chromosome 5

**Challenge**: given a genomic region, how can you find areas of the pangenome are not part of either the CHM13-T2T or GRCh38 reference genomes?

We can calculate the depth (i.e., number of paths crossing each node) via `odgi depth`.  The `-s` parameter can be used to restrict he depth calculations to a subset of paths.  So if we restrict to just the CMH13-T2T and GRCh38 reference genomes, we can find nodes that have a depth of zero (i.e., neither reference crosses the node).  

Get the reference genome path names via:

```{bash}
odgi paths -i chr5.full.og -L | head
```

```
CHM13#chr5
GRCh38#chr5
HG00438#1#JAHBCB010000016.1#0
HG00438#1#JAHBCB010000025.1#0
HG00438#1#JAHBCB010000036.1#0
HG00438#1#JAHBCB010000076.1#0
HG00438#1#JAHBCB010000077.1#0
HG00438#1#JAHBCB010000090.1#0
HG00438#1#JAHBCB010000105.1#0
HG00438#1#JAHBCB010000148.1#0
```

Save the first two to a text file (or just paste them into a text file using the outout from above):

```{bash}
odgi paths -i chr5.full.og -L | head -2 > reference-paths-chr5.txt
```

When identifying novel nodes, we'd also like to know about the size of the node.  The following code extracts the size information for each node:

```{bash}
odgi view -i chr5.full.og -g | grep '^S' | awk -v OFS='\t' '{print($2,length($3))}' | head
```

```
1	2252
2	7748
3	140
4	13
5	3088
6	1
7	1
8	79414
9	923397
10	963697
```

The depth and length information can be combined together using `paste`, and then filtered via `awk` (in this case we are calculating the depth (counting just the two reference paths) per node, and the length per node, then selecting nodes (using `awk`) which have a depth of zero (`$2 < 1`) and a length of greater than 100 (`$5 > 100`). We then print all five columns of information for nodes matching these criteria to a file (`chr5-novel-nodes.txt`):

```{bash}
paste <(odgi depth -i chr5.full.og -s reference-paths-chr5.txt -d | tail -n +2) \
    <(odgi view -i chr5.full.og -g | grep '^S' | \
    awk -v OFS='\t' '{print($2,length($3))}') | awk '{ if ($2 < 1 && $5 > 100) print $1, $2, $3, $4, $5 }' > chr5-novel-nodes.txt
```

```{bash}
head chr5-novel-nodes.txt
```

```
3 0 0 3 140
5 0 0 5 3088
8 0 0 8 79414
9 0 0 9 923397
10 0 0 10 963697
11 0 0 11 995
14 0 0 14 490
15 0 0 15 565
16 0 0 16 1359
17 0 0 17 461
```

To further restrict things, we can select novel nodes that are at least 1000bp long:

```{bash}
cat chr5-novel-nodes.txt | awk '{ if ($5 > 999) print $1, $2, $3, $4, $5 }' > chr5-novel-nodes-1kbp.txt
```

```{bash}
head chr5-novel-nodes-1kbp.txt
```

```
5 0 0 5 3088
8 0 0 8 79414
9 0 0 9 923397
10 0 0 10 963697
16 0 0 16 1359
19 0 0 19 108907
22 0 0 22 1738801
23 0 0 23 3978601
24 0 0 24 1056
25 0 0 25 3715
```

So, we've got a list of nodes that are (1) not present in either CMH13-T2T or GRCh38, and (2) at least 1000bp long.

Next we need to find the depth (i.e., number of haplotypes crossign each node) for each of these.

We can caulcate the depth across *all* nodes on chromosome 5 via:

```{bash}
odgi depth --threads=48 -i chr5.full.og -d > chr5-full-node-depth.txt

head chr5-full-node-depth.txt
```

```
#node.id	depth	depth.uniq
35	1	1
49	1	1
50	1	1
51	1	1
52	1	1
53	1	1
54	1	1
55	1	1
56	1	1
```

We'd like to find novel nodes that appear in a resonable number of haplotypes. Let's arbitrarily choose 10-50 haplotypes:

```{bash}
cat chr5-full-node-depth.txt  | awk '{if ($2 > 9 && $2 < 51) print $1, $2, $3}' > chr5-full-node-depth_ge10_le50.txt

head chr5-full-node-depth_ge10_le50.txt
```

```
3039 13 13
3038 12 12
3042 11 11
3050 12 12
3045 11 11
3057 10 10
3054 12 12
3071 11 11
3076 11 11
3060 10 10
```

That gives as all the nodes on chromosome 5 that involve 10-50 haplotypes.  Now we want to intersect those with the novel nodes identified above.

One way to extract this info is via `grep` - there must be a better way...

First, create a file that will look for lines in the `chr5-full-node-depth_ge10_le50.txt` that start with the numbers of our nodes of interest (adding the `^` symbol denotes the start of the line).

```{bash}
cat chr5-novel-nodes-1kbp.txt | cut -d" " -f1 | sed 's/^/\^/g' > nodes-1kbp.txt

head nodes-1kbp.txt
```

```
^5
^8
^9
^10
^16
^19
^22
^23
^24
^25
```

We can then use that file with `grep` to search for those nodes, using word-based matching (i.e., `-w` in `grep`) - otherwise we'd be matching any line that started with a 5, or an 8, or a 9 etc. We want an extact match: exactly 5 (not 50, or 500 etc).

```{bash}
grep -w -f nodes-1kbp.txt chr5-full-node-depth_ge10_le50.txt > chr5-novel-nodes-ge10-le50-paths.txt

head chr5-novel-nodes-ge10-le50-paths.txt
```

```
77874 22 22
279794 32 32
594873 16 16
594875 16 16
694393 10 10
694447 10 10
694513 10 10
694504 10 10
695721 25 25
695814 25 25
```

These are all of the nodes on chromosome 5 that are:
 - not present in CHM13-T2T or GRCh38
 - at least 1000bp in length
 - involve between 10 and 50 haplotypes

```{bash}
wc -l chr5-novel-nodes-ge10-le50-paths.txt
```

```
79 chr5-novel-nodes-ge10-le50-paths.txt
```

There are 79 or these.

We can now use these node IDs to investigate the characteristics of these nodes.

Reformat to include length info:


```
paste chr5-novel-nodes-ge10-le50-paths.txt \
    <(cat chr5-novel-nodes-ge10-le50-paths.txt | cut -d' ' -f1 | xargs -I {} grep ^{} chr5-novel-nodes-1kbp.txt) |head
```

```
 77874 22 22	 77874 0 0  77874 1037
279794 32 32	279794 0 0 279794 1012
594873 16 16	594873 0 0 594873 2321
594875 16 16	594875 0 0 594875 2200
694393 10 10	694393 0 0 694393 1034
694447 10 10	694447 0 0 694447 1937
694513 10 10	694513 0 0 694513 1039
694504 10 10	694504 0 0 694504 1864
695721 25 25	695721 0 0 695721 1706
695814 25 25	695814 0 0 695814 1008
```

Columns are:

 - Node ID
 - Haplotype depth
 - Unique haplotype depth
 - Node ID
 - Reference depth
 - Unique reference depth
 - Node ID
 - Node length

