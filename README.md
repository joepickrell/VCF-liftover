# `vcf-liftover`, a tool to lift over VCFs without going through BED

# TL;DR: 

## Lifting over a VCF

`./vcf_liftover.sh [chain file] [input.vcf.gz] [output.vcf.gz] [chromosome code] [stop_criterion] [method]`

* `chain file` : chain file that can be downloaded from the USCS website.
* `input.vcf.gz` : input, tabixed bgzipped VCF format.
* `output.vcf.gz` : output, will be bgzipped VCF.
* `chromosome code` : for example, "chr22". This code has to be identical in both the VCF and the chain file.
* `stop_criterion` : if this is `stop` then the liftover file will stop at the first chain in a chromosome. Unless you are overly concerned with performance, write `nostop`.
* `method` : `sort` or `files`. The former consumes lots of memory (and uses PicardTools), the latter creates a lot of files (and uses bcftools). Any other value will just generate a file with intervals and offsets to liftover and exit.

`bcftools` should be in your path. If you use `method=sort`, you must edit the `PICARD=...` line in `vcf-liftover` to use your own version/path of Picard. If you use `method=sort`, be prepared to feed a lot of memory to your process as Picard is incredibly leaky and greedy. 170k variants consume more than 20G of RAM.

A run will always produce a file ending in `.offset`, that follows the `chromosome interval_start interval_end offset`. For every position in these intervals, `offset` must be added to obtain a position in the target build. Regions left unmapped by this file will be lost; they are unmappable or map to a different chromosome.

### Example
```bash
 ~/vcf-liftover/vcf-liftover.sh hg38ToHg19.over.chain 22.vcf.gz 22.liftover.vcf.gz chr22 dontstop files
```

## Principle of a `liftOver`

The USCS LiftOver tool allows to translate genomic coordinates from one build to another. For this, it uses a particular file (called a chain file) that describes how positions in both builds correspond to each other.


The [UCSC chain file format](http://genome.ucsc.edu/goldenPath/help/chain.html) can be a bit hard to understand, all the more so that what is written in a chain file is actually a [net](http://genomewiki.ucsc.edu/index.php/Chains_Nets) comprising of several hierarchically arranged chains.

In short, chains are sequences of alignments separated by gaps. So a chain file is easily translatable into a series of genomic intervals with an attached offset. Every SNP located within one such interval will need to be shifted by the corresponding offset. We first write a code snippet that produces a list of intervals and offsets from a chain file.

## Making sense of chain files

The following piece of perl code finds the first chain that describes any given chromosome (as said before, there are many chains per chromosome, and the first, being the one with the highest score, usually spans its entire length). Every ungapped alignment that follows is then translated into an interval.

```bash
cat $CHAINFILE | perl -lane '
    if($F[0] eq "chain"){
          if($F[2] ne $F[7] || $F[2] ne $ENV{"CHR"}){
                 $skip=1;
          }
          else{$curchr=$F[2];our $current=$F[5];our $coffset=$F[10]-$F[5];$skip=0;}
    }
elsif(!$skip) {
	my $offset=$F[2]-$F[1];
	print "$curchr $current ", $current+$F[0], " $coffset"; 
	$current=$current+$F[0]+$F[1];
	$coffset=$coffset+$offset;
}
' 
```

The code above translates this chain file excerpt:

```
chain 3231099988 chr22 50818468 + 16367188 50806138 chr22 51304566 + 16847850 51244566 23
19744   0       40
36      1       1
```

into the following list of intervals:

```
chr22 16367188 16386932 480662
chr22 16386932 16386968 480702
chr22 16386969 16387000 480702
```

and so forth for all chromosomes. Theoretically, we should descend into further chains instead of using only the first one, as those will further describe mappings within gaps left blank by the topmost chain. 

## Lifting over coordinates in a VCF

What we do next is a bit inefficient: given a chromosome-wide VCF file, we loop over all intervals in the modified chain file, tabixing them out of the VCF and applying the desired offset.

```bash
{ 
tabix -h $INFILE 0:0-1 && cat $OFFSETS | grep $CHR | while read line;  do 
	IFS=" " read -r -a fields <<< "$line"
	tabix $INFILE ${fields[0]}:${fields[1]}-${fields[2]} | awk -v offset=${fields[3]} 'BEGIN{OFS="\t"}{$2=$2+offset;}1'; 
done 
} | bgzip > $OUTFILE
```

## Performance of liftover
### Speed
`vcf-liftover` converts 175k variants in about 
* 20 minutes with the `files` method
* 13 minutes with the `sort` method

### Accuracy 
* `vcf-liftover` ignores intervals that map on different chromosomes
* and intervals that map from a positive strand to a negative one.

 Other than that, lifted over positions are identical between the two tools.

