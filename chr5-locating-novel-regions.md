## Locating novel chromosome 5 regions in the GRCh38 reference genome

Once novel regions have been identified, it is useful to know where these are relative to anexisting reference genome (i.e., which nodes in the reference genome are adjacent to the novel nodes?)

Extract path name for GRCh38 (or chromosome 5)

```{bash}
odgi paths -i chr5.full.og -L | head -2 | tail -1 > grch38-path-name-chr5.txt
more grch38-path-name-chr5.txt
```

```
GRCh38#chr5
```

```
# Extract the nodes for just the GRCh38 path
odgi extract --threads=48 -i chr5.full.og --paths-to-extract=grch38-path-name-chr5.txt -r GRCh38#chr5 -o chr5-grch38.og

# Get the node IDs that are present in the GRCh38 path
odgi view -i chr5-grch38.og -g | grep ^S | cut -d$'\t' -f2 > chr5-grch38-node-list.txt

# Highest node ID BEFORE the specified NRS node (77874)
cat chr5-grch38-node-list.txt | awk '{if($1 < 77874) print $1}' | tail -1



  
