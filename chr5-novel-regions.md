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

Save the first two to a text file (or just paste them into a text file using the output from above):

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

Which paths go through a specific node?

Extract node data (first node from list above):

```{bash}
odgi extract -i chr5.full.og -n 77874 -o node77874.og
```

Node stats:

```{bash}
odgi stats -i node77874.og -S
```

```
#length	nodes	edges	paths	steps
   1037	    1	    0	   22	   22
```

Paths passing through node:

```{bash}
odgi paths -i node77874.og -L
```

```
HG00621#1#JAHBCD010000062.1#0:47002952-47003989
HG00735#1#JAHBCH010000026.1#0:952592-953629
HG01106#2#JAHAMB010000075.1#0:960088-961125
HG01175#2#JAHALZ010000034.1#0:941000-942037
HG01358#1#JAGYZB010000159.1#0:863608-864645
HG01891#1#JAGYVO010000064.1#0:948347-949384
HG02055#2#JAHEPJ010000067.1#0:3440418-3441455
HG02145#2#JAHKSF010000135.1#0:3443959-3444996
HG02148#1#JAHAMG010000102.1#0:2529704-2530741
HG02257#2#JAGYVH010000030.1#0:1047631-1048668
HG02486#1#JAGYVM010000093.1#0:957382-958419
HG02622#2#JAHAON010000092.1#0:961627-962664
HG02717#1#JAHAOS010000269.1#0:74085-75122
HG02717#2#JAHAOR010000026.1#0:951301-952338
HG02723#1#JAHEOU010000195.1#0:849625-850662
HG02723#2#JAHEOT010000155.1#0:954122-955159
HG02886#2#JAHAOT010000202.1#0:2523845-2524882
HG03492#1#JAHEPI010000152.1#0:536356-537393
HG03540#1#JAGYVY010000270.1#0:25409-26446
HG03540#2#JAGYVX010000142.1#0:3441959-3442996
NA18906#2#JAHEON010000113.1#0:5829706-5830743
NA20129#2#JAHEPD010000252.1#0:945117-946154
```

Extract sample IDs to file:

```{bash}
odgi paths -i node77874.og -L | cut -d"#" -f1 > node77874-samples.txt
```

In R:

```{r}
# Load dplyr package
library(dplyr)

# Read in the node sample data
x = readLines('node77874-samples.txt')

length(x)
```

```
22
```

```{r}
length(unique(x))
```

```
19
```

Must be some homozygotes.

```{r}
table(x) %>% sort()
```

```
HG00621 HG00735 HG01106 HG01175 HG01358 HG01891 HG02055 HG02145 HG02148 HG02257 
      1       1       1       1       1       1       1       1       1       1 
HG02486 HG02622 HG02886 HG03492 NA18906 NA20129 HG02717 HG02723 HG03540 
      1       1       1       1       1       1       2       2       2
```

Load kgp package, which contains sample metadata from 1000 Genomes Project:

```{r}
library(kgp)
```

Use `kgpe` object (pedigree and population information all 3,202 samples
included in the expanded 1000 Genomes Project data, which includes 602 trios).

```{r}
data(kgp)
data.frame(kgpe[match(x, kgpe$id),] )
```

```
     fid      id     pid     mid sex   sexf  pop  reg                              population     region phase3
1  SH066 HG00621 HG00619 HG00620   1   male  CHS  EAS             Southern Han Chinese, China  East Asia  FALSE
2   PR06 HG00735 HG01047 HG00734   2 female  PUR  AMR             Puerto Rican in Puerto Rico    America  FALSE
3   PR25 HG01106 HG01104 HG01105   1   male  PUR  AMR             Puerto Rican in Puerto Rico    America  FALSE
4   PR36 HG01175 HG01173 HG01174   2 female  PUR  AMR             Puerto Rican in Puerto Rico    America  FALSE
5  CLM31 HG01358 HG01356 HG01357   1   male  CLM  AMR         Colombian in Medellin, Colombia    America  FALSE
6   BB05 HG01891 HG01890 HG01889   2 female  ACB  AFR           African Caribbean in Barbados     Africa  FALSE
7   BB15 HG02055 HG02053 HG02054   1   male  ACB  AFR           African Caribbean in Barbados     Africa  FALSE
8   BB20 HG02145 HG02143 HG02144   1   male  ACB  AFR           African Caribbean in Barbados     Africa  FALSE
9  PEL39 HG02148 HG02146 HG02147   2 female  PEL  AMR                  Peruvian in Lima, Peru    America  FALSE
10  BB21 HG02257 HG02255 HG02256   2 female  ACB  AFR           African Caribbean in Barbados     Africa  FALSE
11  <NA>    <NA>    <NA>    <NA>  NA   <NA> <NA> <NA>                                    <NA>       <NA>     NA
12  GB31 HG02622 HG02620 HG02621   2 female  GWD  AFR Gambian in Western Division, The Gambia     Africa  FALSE
13  GB50 HG02717 HG02715 HG02716   1   male  GWD  AFR Gambian in Western Division, The Gambia     Africa  FALSE
14  GB50 HG02717 HG02715 HG02716   1   male  GWD  AFR Gambian in Western Division, The Gambia     Africa  FALSE
15  GB52 HG02723 HG02721 HG02722   2 female  GWD  AFR Gambian in Western Division, The Gambia     Africa  FALSE
16  GB52 HG02723 HG02721 HG02722   2 female  GWD  AFR Gambian in Western Division, The Gambia     Africa  FALSE
17  GB89 HG02886 HG02884 HG02885   2 female  GWD  AFR Gambian in Western Division, The Gambia     Africa  FALSE
18  PK43 HG03492 HG03490 HG03491   1   male  PJL  SAS             Punjabi in Lahore, Pakistan South Asia  FALSE
19 GB125 HG03540 HG03538 HG03539   2 female  GWD  AFR Gambian in Western Division, The Gambia     Africa  FALSE
20 GB125 HG03540 HG03538 HG03539   2 female  GWD  AFR Gambian in Western Division, The Gambia     Africa  FALSE
21  Y020 NA18906 NA18877 NA18876   2 female  YRI  AFR               Yoruba in Ibadan, Nigeria     Africa  FALSE
22  2433 NA20129 NA19920 NA19921   2 female  ASW  AFR        African Ancestry in Southwest US     Africa  FALSE
```

Samples by population:

```{r}
kgpe$pop[match(x, kgpe$id)] %>% table() %>% sort()
```

```
ASW CHS CLM PEL PJL YRI PUR ACB GWD 
  1   1   1   1   1   1   3   4   8
```

Samples by region (i.e., super-population): 

```{r}
kgpe$region[match(x, kgpe$id)] %>% table() %>% sort()
```

```
 East Asia South Asia    America     Africa 
         1          1          5         14
```
