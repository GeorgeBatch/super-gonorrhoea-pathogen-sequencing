# Health Data Science CDT - Tutorial: Pathogen sequencing 

## Detecting antimicrobial resistance determinants

### Overview
In this tutorial you will learn how to use various scriptable approaches to detect different antimicrobial resistance determinants and use sequencing data to assess the extent of transmission of infection in hospitals. The tutorial will be mostly based using Python, but also rely on additional software packages. Please produce your answers as an annotated Python jupyter notebook.

Files for Tasks 1-4 can be found in the `super_gonorrhoea` folder and files for Task 5 in the `hospital_transmission` folder.


### Background
As discussed in the lecture, antimicrobial resistance can be determined by several mechanisms
* Modification of the target
* Degradation of the antimicrobial
* Reduced influx or increased efflux

In turn each mechanism can be brought about by several changes at the genome level
* Single nucleotide polymorphisms (SNPs) in genes or their regulators
* Small insertions or deletions - indels in genes or their regulators
* Gain of new genes on mobile genetic elements including plasmids

Three main apporaches can be used to look for antimicrobial resistance determinants
* Reference based mapping - good for SNPs and some indels, but can't detect novel genetic material like gain of a plasmid
* De novo assembly - good for gain of novel material and larger insertions and deletions

Neither approach works well where there are multiple copies of a gene in a genome - we will develop a bespoke approach for this.


### The world's worst super gonorrhoea
In February 2018, a patient was diagnosed with the most difficult to treat gonorrhoea ever described. You can read more about it here - https://www.bbc.co.uk/news/health-43571120 - and his suscessful treatment here - https://www.bbc.co.uk/news/health-43840505. The more scientific version is here - https://www.eurosurveillance.org/content/10.2807/1560-7917.ES.2018.23.27.1800323 - but I would encourage not to look at this too closely to start with, as it contains many of the answers!

Two isolates from this infection were sequenced using Illumina technology. You are provided with one example, including the following files:
* Sequence reads - the raw output of the sequencer - these are too large to include in this repository and can be downloaded from http://ftp.sra.ebi.ac.uk/vol1/fastq/ERR256/009/ERR2560139/ERR2560139_1.fastq.gz and here - http://ftp.sra.ebi.ac.uk/vol1/fastq/ERR256/009/ERR2560139/ERR2560139_2.fastq.gz (alternatively if running this practical during the course - please copy from `/cdtshared/wgs_practical`)
* A ready-made mapped concensus fasta file - `super_gc_mapped.fa`
* A ready-made de novo assembly `super_gc_contigs.fa`
* A copy of the reference genome to be used for comparisons `reference.fa`

The mapped file was generated by mapping reads to a reference genome (e.g. using BWA mem), and searching these for variants from the reference genome (e.g. using samtools mpileup), and quality filtering them to identify variants of high quality. The de novo assembly was made an assembler (e.g. SPAdes).

The infeciton was caused by a gonorrhoea isolate that was resistant to multiple antibiotics - we will attempt to explain these

Antimicrobial|MIC|Interpretation
:--- | :--- | :----
Ceftriaxone	|0.5 mg/L	|Resistant
Cefixime	|2 mg/L	|Resistant
Azithromycin	|> 256 mg/L	|High-level resistant
Ciprofloxacin	|> 32 mg/L	|Resistant
Tetracycline	|32 mg/L	|Resistant
Benzylpenicillin	|1 mg/L	|intermediate susceptible
Spectinomycin	|8 mg/L	|Susceptible
Gentamicin	|2 mg/L	|No resistance breakpoint available (low value)
Ertapenem	|0.032 mg/L	|No resistance breakpoint available (low value)

MIC, minimum inhibitory concentration.

### Suggested resources
You will need to use the following biopython tools in the tutorial
* https://biopython.org/wiki/SeqIO - for reading in the mapped files
* http://biopython.org/DIST/docs/tutorial/Tutorial.html#htoc93 - for searching de novo assemblies

You will also need a way to re-map the raw reads to a novel reference
* You can do this with BWA mem - https://github.com/lh3/bwa
* You will need a way to analyse the mapped files - I suggest using pysam to run samtools - https://pysam.readthedocs.io/en/latest/api.html#introduction

Most of the relevant resistance mutations and variants in Neisseria gonorrhoeae (the bacteria that causes gonorrhoea) are catelogued in the NG-STAR database - https://ngstar.canada.ca/alleles/query?lang=en

If you want to know more about the ways in which gonorrhoea can become resistance to commonly used antibiotics this is a good review - https://cmr.asm.org/content/27/3/587.long
<br />
<br />

## Task 1 - Explain ciprofloxacin resistance
**A common mutation causing ciprofloxacin resistance is found in the *gyrA* and results in the following amino acid change - S91F. Can you confirm if this is present in the sequence data, using the mapped data, and find another resistance determinant?**

This relies on using python to identify the gene in the whole genome andconvert DNA sequence to protein sequences, so we can then compare the reference genome with our genome of interest to look for differences.

Tips
* Find the location of the *gyrA* gene in the reference file using the NCBI website - https://www.ncbi.nlm.nih.gov/nuccore/NC_011035.1?report=graph - use the find function on the webpage
* Use biopython's `SeqIO.read()` function to read in the mapped fasta file `super_gc_mapped.fa` (https://biopython.org/wiki/SeqIO)
* You can will need to extract the DNA sequence of the gene using it's coordinates within the whole genome (remember that these number from zero in python, but from one on the NCBI website!)
* You will need to translate the DNA sequences to a protein sequence, `seq.translate()` in biopython (for *gyrA* the gene is in the same direction as the DNA is numbered, if it were not you would need to generate the reverse complement sequence first), if this works you should end up with a string of amino acids represented as single letters and ending with a stop codon shown as an asterisk.
* Compare the amino acid sequences from the reference with the sequence from the case using python, where are the differenes and what are they?
<br />
<br />

## Task 2 - Explain the tetracycline resistance
**Tetracycline resistance is conferred by gain of a new gene, *tetM*, carried on a plasmid. Is this gene present?**  
To do this we will use a tool called BLAST. This is an optimised algorithm for searching for one or more sequences (the query) within a database of sequences. In this example we will treat the gene we are trying to find as the query and the database will be the contigs that make up the assembly of our genome of interest.

I suggest you read the Process and Algorithm sections of the wikipedia article on BLAST before continuing - as this will help with parsing the output generated - https://en.wikipedia.org/wiki/BLAST_(biotechnology).

Tips
* Use the de novo assembly provided `super_gc_contigs.fa`
* You will need to create blast database files for the de novo assembly with this command:  
```makeblastdb -dbtype nucl -in super_gc_contigs.fa```
* Use the biopython blast module to run a blast command that searches for the tetM gene, to get you started:  
```
from Bio.Blast import NCBIXML
from Bio.Blast.Applications import NcbiblastnCommandline
import StringIO

geneFile = 'tetM.fa'
assemblyFile = 'super_gc_contigs.fa'

cline = NcbiblastnCommandline( query=geneFile, 
		db=assemblyFile, 
		evalue=0.01, 
		outfmt=5
		)
stdout, stderr = cline()
fp = StringIO.StringIO( stdout )
blast_records = NCBIXML.parse( fp )
```
This snippet will run blast, saving the output in XML format, this is then passed back to python. You should be able interrogate the `blast_records` object to look for any matches found. Details of the contents of the object can be found here - http://biopython.org/DIST/docs/tutorial/Tutorial.html#htoc93.
* Did you find an alignment match? How much of the total length of the tetM gene was matched? How similar was it to the tetM sequence in `tetM.fa`?

<br />
<br />

## Task 3 - Explain the resistance to ceftriaxone
**Here we build on the example above to look for the best match to the *penA* gene using BLAST.**  
A number of mechanisms can change how susceptible N. gonorrhoeae is to ceftriaxone, however the gene *penA* is the most important. Although a few SNPs within the *penA* gene, e.g. A311V, T316P and T483S, the rest of the gene is very diverse (it was generated by recombination events with similar genes in other Neisseria species), and so mapping approaches do not work well for finding these variants - as the sequence is too different to the reference to map reliably. We therefore use a de novo assembly and a database of know *penA* sequences or alleles. We will compare our sequence of interest to this list of alleles. 

Tips
* Start as in Task 2, using the assembly as the blast database and this time using the list of *penA* alleles as the query `penA_alleles.fa` (if you want you can also try this task in reverse using the list of *penA* alleles as the database, to do this you'll need to run makeblastdb on the command line as above)
* You will get multiple hits, you need to define which hit is the closest match
* Once you find which hit is the closest match, check to see if there is an exact match, and translate the sequence to see which of these amino acid substitutions is present - A311V, T316P and T483S

<br />
<br />

## Task 4 - Explain resistance to azithromycin
**This will require a completely novel approach as there are 4 copies of the 23S RNA gene in the genome**
Why do you think that neither whole genome mapping or de novo assembly work here? Can you think of why this might cause problems in other situations, e.g. where a bacteria acquires multiple copies of very similar genes on one or more plasmids?

In this case we need to search the genome for 4 identical copies of the 23S rRNA genes. As the proportion of the 4 genes with a particular mutation goes up, the level of resistance present goes up too. We need to search for 2 mutations in the DNA sequence A2059G and C2611T, however the numbering convention here is based on a different version of the gene, and so these sites correspond to positions 2045 and 2597 in `23s_reference.fa`, numbered starting from 1.

To determine how many copies of the mutation are present we will map all the read to a single copy of the 23S rRNA gene, and then count the proportion of the reads with the mutation and approximate this to the nearest 25%, i.e. 0, 1, 2, 3, or 4 copies, based on 0, 25, 50, 75, 100% of reads.

Tips
* to map the reads to the `23s_reference.fa` using BWA you'll need to prepare the file to be used with `bwa mem`:  
`bwa index 23s_reference.fa`  
It may be worth doing this within the python notebook, so you can track this too as part of your answer, e.g.  
```
cmd = '%s/bwa index %s'%(bwaPath, ref)
print cmd
os.system(cmd)
```
* now map the reads using something based on this snippet, where fq1 and fq2 are the two files downloaded from the short read archive above
```
cmd = '%s/bwa mem -t 4 %s %s %s > %s'%(bwaPath, ref, fq1, fq2, out_sam)
print cmd
os.system(cmd)
```  
This will take a few minutes to run
* Now put the mapped reads into a sorted BAM file and index it using samtools
```
# put the mapped reads (-F 4) into sorted Bam
cmd = 'samtools view -uS -F 4 %s | samtools sort - -o %s'%(out_sam, out_bam)
print cmd
os.system(cmd)

cmd = 'samtools index %s'%out_bam
print cmd
os.system(cmd)
```
* now use pysam to check what proportion of reads contain the variant, using a construct starting similar to this...
```
samfile = pysam.Samfile(out_bam, "rb")
for pileupcolumn in samfile.pileup( '23s_rna', 2000, 2600):
	for pileupread in pileupcolumn.pileups:
```
* How many copies of each SNP is present? Does this explain the high level of azithromycin resistance seen?

<br />
<br />

## Task 5 - Something different...
You now have the tools that would enable you to look for other variants seen commonly in in *N. gonorrhoeae*. If you have finished with lots of time to spare you could you the allele files that you can download from https://ngstar.canada.ca/alleles/loci_selection?lang=en to look for variants in *mtrR*, *porB*, *ponA*, and *parC*. You could also check you answer to Task 1, by using the *gryA* alleles and a BLAST based approach.

However I suggest you now move on to a different task. I'm assuming you have already spent time creating phylogenetic trees in a previous practical. You could apply those approaches to gonorrhoea here and try to find the closest matching genomes to work out where the super gonorrhoea we have been investigating might have come from. However, we are going to change to another bacteria, *Clostridium difficile*. *C. difficile* commonly causes diarrhoea in patients admitted to hospital. 

**You will look at a summary of *C. difficile* sequence data from six hospitals and try to work out which hospital has the most transmission.**

This is a less structured task than those above, you are provided with two files
* `sample_list.csv` which contains a list of 973 samples, each from one of six sites or hospitals, you are provided with collection dates, rt (the samples ribotype - a way of grouping *C. difficile*), the fecal_tox column tells you if POS then the patient has disease and if NEG then they are a carrier, the toxigenic column tells you whether this strain has toxin genes present
* `pairwise_snps.csv` contains the pairwise genetic distances (SNP differences) between all samples obtained from a maximum likelihood phylogeny corrected for the effect of recombindation, using a tool called ClonalFrameML

Generally speaking cases acquired from each other are within ≤2 SNPs >95% of the time.

For this investigation you will need to ignore non-toxigenic cases.

Try to answer the following
1. Which hospital has the most transmission?
2. Which are more infectious - fecal_tox positive or negative cases?
3. Is there evidence of transmission between hospitals?
4. How big is each transmission cluster?

Again I would encourage you not to cheat - but the paper we wrote on this dataset can be found here for those who would like to read more - https://academic.oup.com/cid/article/65/3/433/3857742.

<br />
<br />

### Required / recommended software if trying this elsewhere
* Python, with jupyter notebook, biopython, pysam, numpy, pandas and networkx libraries
* BLAST
* BWA
* samtools

<br />
David Eyre<br />
26 February 2020<br />
david.eyre@bdi.ox.ac.uk<br />
