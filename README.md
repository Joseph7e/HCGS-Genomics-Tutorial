# HCGS-Genomics-Tutorial

Bacterial Genome Assembly and Assessment Tutorial

**This site will be continuously updated throughout the week.**
[Go to where we left off](#blast-the-entire-assembly-against-the-nt-database)

### General Notes:
**For each program that we run in this tutorial I have provided a link to the manual**. These manuals provide a thorough explanation of what exactly we are doing. Before running the program it is a good idea to skim through these, examine the options, and see what it does. It is also a good idea to check out the publication associated with the program. Please note that the commands we run are general and are usually executed with default settings. This works great for most genomes but the options may need to be tweaked depending on your genome. Before you run any command it is also a great idea to look at the programs help menu. This can usually be done with the name of the program followed by '-h' or '-help' or '--help'. i.e. 'spades -h'. Also ... never forget about google for quick answers to any confusion.

This tutorial assumes a general understanding of the BASH environment. **You should be familiar with moving around the directories and understand how to manipulate files**.

See the BASH tutorial to get started.

https://github.com/Joseph7e/HCGS-BASH-tutorial

Throughout this tutorial the commands you will type are formatted into the gray text boxes (don't do it when learning but they can be faithfully copied and pasted). The '#' symbol indicates a comment, BASH knows to ignore these lines. 

**Remember to tab complete!** There is a reason the tab is my favorite key. It prevents spelling errors and allows you to work much faster. Remember if a filename isn't auto-completing you can hit tab twice to see your files while you continue typing your command. If a file doesn't auto-complete it means you either have a spelling mistake, are in a different directory than you originally thought, or that it doesn't exist.


## Starting Data:

Isolate Bacteria            |  Extract DNA
:-------------------------:|:-------------------------:
![alt text](https://github.com/Joseph7e/HCGS-Genomics-Tutorial/blob/master/petri.jpg?raw=true)  |  <img src="https://www.cephamls.com/wp-content/uploads/2019/02/DNA-Extraction-Figure-3-22.jpg" width="420">
<img src="https://github.com/Joseph7e/HCGS-Genomics-Tutorial/blob/master/fragmentation3.png?raw=true" width="800">

Prepare Library           |  Sequence DNA
:-------------------------:|:-------------------------:
<img src="https://jef.works//assets/blog/librarystructure.png" width="520">  |  <img src="https://github.com/Joseph7e/HCGS-Genomics-Tutorial/blob/master/hiseq.png?raw=true" width="320">


Your starting data is found within a shared directory within your group folder (one directory level up). To start we will move a set of Sample data into your home directories. Each of these samples represent the genome of a unique and novel microbe that has not been seen before (except by me). Inside this directory are Illumina HiSeq 2500, paired-end, 250 bp sequencing reads. Looking in this directory you should see two files per sample, the forward and reverse reads. These files are in **FASTQ** format (see below).

```bash
# Copy a sample from the shared directory to workshop folder, 
# USE AUTOCOMPLETE

#make sure you start in the home dir
cd
mkdir Sample_data
cd Sample_data/

# copy data, be sure to replace SAMPLE# with your data
cp -r /home/genome/joseph7e/workshop_data_june22nd/remembertoautocompletecommands/raw-reads/*_SAMPLE#_* ./
# confirm the copy arrived (remember ‘*’ will match any character/string)
ls

# go back home
```

[Link explaining the 'Read Name Format'](http://support.illumina.com/content/dam/illumina-support/help/BaseSpaceHelp_v2/Content/Vault/Informatics/Sequencing_Analysis/BS/swSEQ_mBS_FASTQFiles.htm): SampleName_Barcode_LaneNumber_001.fastq.gz


Important note: In the above command I use the "\*" character to view the Sample directory, I would normally just type out the entire path using tab complete (which is what you should do). This wildcard will match any string of characters. I use this because everyone will have a different Sample name. To make this tutorial as general as possible I need to use these wildcards throughout the tutorial. In addition I may use Sample_X instead of Sample_\*. In these cases be sure to type out your complete sample name!, the wildcards probably won't work 


* Prepare your working directory

It is a good idea to keep your directories tidy and to name your files something that makes sence. This is just to keep things organized so you know what everything is several months from now. We are going to make a new directory to house all of the analyses for this tutorial.

```bash
# Make a new directory
mkdir genomics-tutorial/

# move the sample data into the directory
mv Sample_data/ genomics-tutorial/

# change into the directory
cd genomics-tutorial/

# make the sample directory name more meaningful
mv Sample_X Sample_X-raw_reads
```

## Examine the Raw Reads

Note the file extension - fastq.**gz**. Since these files are usually pretty big it is standard to receive them compressed. To view these files ourselves (which you normally wouldn't do) you either have to decompress the data with gzip or by using variations of the typical commands. Instead of 'cat' we use 'zcat', instead of grep we can use 'zgrep'. Below I show both ways.
       
```bash
# Examine the reads with zcat, I use the wildcard '*' to match the file since everyone's names will be different. Use tab complete and there is no need for the wildcards.
zcat Sample*/*_R1_* | more
# unzip the data and view
gunzip Sample*/*_R1_*
more Sample*/*_R1_*
# rezip the data
gzip Sample*/*_R1_*
# Examine reads directly with less
less -S Sample*/*_R1_*
```

* Fastq File Format - each sequencing read entry is four lines long.. 

    - Line 1. Always begins with an '@' symbol and denotes the header. This is unique to each sequence and has info about the sequencing run. 

    - Line 2. The actual sequencing read for your organism, a 250 bp string of As, Ts, Cs, and Gs.

    - Line 3. Begins with a '+' symbol, this is the header for the read quality. Usually the same as the first line header. 

    - Line 4. Next are ascii symbols representing the quality score (see table below) for each base in your sequence. This denotes how confident we are in the base call for each respective nucleotide. This line is the same length as the sequencing line since we have a quality score for each and every base of the sequence. 

![rawilluminadatafastqfiles](https://user-images.githubusercontent.com/18738632/42129269-49b8dace-7c8e-11e8-86e7-069df9028447.png)

![quality_info](https://user-images.githubusercontent.com/18738632/42226531-2f343178-7ead-11e8-8401-5a2fb455b4ef.png)

* Count The Number of Raw Reads

I always start by counting the number of reads I have for each sample. This is done to quickly assess whether we have enough data to assemble a meaningful genome. Usually these file contains millions of reads, good thing BASH is great for parsing large files! Note that the forward and reverse reads will have the same number of entries so you only need to count one.

```bash
# using grep. Note that I don't count just '@', this is because that symbol may appear in the quality lines.
zgrep -c '^@' Sample*/*R1*
# counting the lines and dividing by 4. Remember each read entry is exactly four lines long. These numbers should match.
zcat Sample*/*_R1_* | wc -l
```
* Whats our total bp of data? This is what we call our sequencing throughput. We multiple the number of reads by the read length (ours is 250) and by 2 because it is paired-end data.

(Read length x 2(paired-end) x Number of reads)

```
# we can do this calculation from the terminal with echo and bc (bc is the terminal calculator)
echo "Number_of_reads * 250 * 2" | bc
```

* If we have a 7 MB genome, what is our average coverage? 

(Total bp/7,000,000)
```
echo "Total_bps / 7000000" | bc
```

If you completed the above calculation lets hope you have at least 10X coverage. For the most part, the higher the coverage the better off we are. If you have low coverage you'll want to do some more sequencing and get more read data. Usually published genomes have at least 70-100X coverage.

## Read Quality Check w/ FASTQC
manual: https://www.bioinformatics.babraham.ac.uk/projects/fastqc/

[FASTQC explained](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/Help/3%20Analysis%20Modules/)

* Run Fastqc

FastQC is a program to summarize read qualities and base composition. Since we have millions of reads there is no practical way to do this by hand. We call the program to parse through the fastq files and do the hard work for us. **The input to the program is one or more fastq file(s) and the output is an html file with several figures.** The link above describes what each of the output figures are describing. I mainly focus on the first graph which visualizes our average read qualities and the last figure which shows the adapter content. Note that this program does not do anything to your data, as with the majority of the assessment tools, it merely reads it.

```bash
# make a directory to store the output
mkdir fastqc_raw-reads
# run the program
fastqc Sample_*/*_R1_* Sample_*/*_R2_* -o fastqc_raw-reads
ls fastqc_raw-reads
# the resulting folder should contain a zipped archive and an html file, we can ignore the zipped archive which is redundant.
```

* Transfer resulting HTML files to computer using filezilla or with the command line on OSX/Linux.

With Cyberduck you will need to enter the same server information when you login form the terminal. Be sure to use port 22.  

```bash
# to get the absolute path to a file you can use the ‘readlink’ command.
readlink -f fastqc_raw-reads/*.html
# copy those paths, we will use them for the file transfer
# In a fresh terminal on OSX, Linux, or BASH for windows
scp USERNAME@ron.sr.unh.edu:/home/GROUP/USERNAME/mdibl-t3-2019-WGS/fastqc_raw-reads/*.html /path/to/put/files
```

* Transfer resulting HTML files to computer using filezilla or with the command line on OSX/Linux.

On filezilla you will need to enter the same server information when you login form the terminal. Be sure to use port 22.  

```bash
# to get the absolute path to a file you can use the ‘readlink’ command.
readlink -f fastqc_raw-reads/*.html
# copy those paths, we will use them for the file transfer
# In a fresh terminal on OSX, Linux, or BASH for windows
scp USERNAME@ron.sr.unh.edu:/home/GROUP/USERNAME/mdibl-t3-2019-WGS/fastqc_raw-reads/*.html /path/to/put/files
```

## Adapter and Quality Trimming w/ Trimmomatic
manual: http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/TrimmomaticManual_V0.32.pdf

alternative tools: [cutadapt](http://cutadapt.readthedocs.io/en/stable/guide.html), [skewer](https://github.com/relipmoc/skewer)

* Run Trimmomatic

You may have noticed from the fastqc output the some of your reads have poor qualities towards the end of the sequence, this is especially true for the reverse reads and is common for Illumina data. You may also notice that the fastqc report 'failed' for adapter content. The Trimmomtic program will be used to trim these low quality bases and to remove the adapters. I created a wrapper script called trim_script_TruSeq.sh which makes this program much easier to use. It is available on the server by calling its name, it is also available on this github repository. For this wrapper script **the input is the raw forward and reverse reads and the output will be new trimmed fastq files**. We will use these trimmed reads for our genome assembly. When you are more comfortable using BASH you can call Trimmomatic directly by using the manual or by copying the code from the provided script.

```bash
# Run wrapper script
trim_script_TruSeq.sh Sample_*/*_R1_* Sample_*/*_R2_*
# if you want to see inside the program you can take a look.
which trim_script_TruSeq.sh
more /usr/local/bin/trim_script_TruSeq.sh
```

* Move the trimmed reads to new directory - remember its a good idea to keep the directory clean.
```bash
mkdir trimmed_reads
# move all the reads into the new directory
mv *fastq.gz trimmed_reads/
# confirm that the files have moved
ls trimmed_reads/
```

When the program finishes it outputs four files. paired_forward.fastq.gz, paired_reverse.fastq.gz, and two unpaired reads. These output files are cleaned reads, which hopefully retained the highly confident sequences and have removed the adapters from the sequences. Some sequences will be lost entirely, some will lose a few bases off the ends, and some won't be trimmed at all. When a reverse read is lost but a forward read is maintained, the forward read will be written to the unpaired_forward.fastq.gz file (and vise-versa).

![fastqc](https://user-images.githubusercontent.com/18738632/42241259-ef2d5f0c-7ed7-11e8-8a7f-f7407979202f.png)

Similar to above, you can run FASTQC again with your new trimmed reads. Comparing the original html and the new one you should note the differences (see above).

You can also count the number of reads for each of your files like you did for the raw reads. How does this compare to the original count? What percentage of your reads did you lose? How many reads are unpaired?

## Genome Assembly

<img src="https://www.genome.gov/sites/default/files/tg/en/illustration/shotgun_sequencing.jpg">


manual: http://cab.spbu.ru/software/spades/

alternative tools: [ABySS](http://www.bcgsc.ca/platform/bioinfo/software/abyss), [MaSuRCA](http://masurca.blogspot.com/)

With our trimmed reads in hand are now ready to assemble our genomes (check out Kelley's PowerPount for how it works). There are many programs that are used for genome assembly and different assemblers work well with certain genomes (how large the genome is, how complex, is it a Eukaryote, etc), but SPAdes works very well for most bacteria. Either way these programs are usually run with the same sort of syntax. **The input is a set of sequencing reads in FASTQ format and the output will be a FASTA file which is the genome assembly**. I encourage you to try out a different assembler and compare the results.

* Run SPAdes
```bash
# examine the help menu
spades.py --help
# run spades with the forward, reverse, and unpaired reads.
nohup spades.py -1 trimmed_reads/paired_forward.fastq.gz -2 trimmed_reads/paired_reverse.fastq.gz -s trimmed_reads/unpaired_forward.fastq.gz -s trimmed_reads/unpaired_reverse.fastq.gz -o spades_assembly_default -t 24 &
```

Notice that the above command makes use of 'nohup' and '&'. Its good practice to always use these two together. This allows you to close your computer and let the server continue working and/or let you continue working while the job runs in the background. This is the most computationally expensive program of the pipeline. It is taking our millions of reads and attempting to put them back together, as you can image that takes a lot of work. It will probably take a few hours to run.

You can check the output of nohup for any errors. If there are any errors you will see them at the bottom of the file. The best way to check is with the 'tail' command. Try 'tail nohup.out'.

* Check the status of your job w/ 'top' -- [top explanation](https://www.booleanworld.com/guide-linux-top-command/)
```bash
# 'top' lets you see all the jobs that are running on the server
top
# adding the user option lets you view just your jobs
top -u $USER
# the "$USER" variable always links to your user name, to see what I mean echo it.
echo $USER
```

* View Output Data When the assembly finishes.

```bash
ls spades_assembly_default/
# view FASTA file
less -S spades_assembly_default/contigs.fasta
# view the top 10 headers
grep '>' spades_assembly_default/contigs.fasta | head
# count the number of sequences
grep -c '>' spades_assembly_default/contigs.fasta
```

## FASTA format

The FASTA format is similar to the FASTQ format except it does not include quality information. Each sequence is also delineated by a '>' symbol instead of a '@'. In addition, all the sequences will be much larger (since they were assembled). Instead of all the sequencing being 250 bp they could be the size of an entire genome!  Each of the sequence entries in the FASTA file are typically refereed to as a contig, which means contiguous sequence. In an ideal world the assembler would work perfectly and we would have one contig per chromosome in the genome. In the case of a typical bacterium (if there is such a thing) this would mean one circular chromosome and maybe a plasmid. So if the assembly worked perfect we would see two contigs in our FASTA file. However, this is very rarely the case (unless we add some sort of long-read technology like pacbio or nanopore sequencing). How fragmented your reconstructed genome is usually depends on how many reads you put into your assembler, how large the genome is, and what the architecture and complexity of the genome is like. We typically see a genome split into 10's to 100's of contigs for a typical run.

In the case of SPAdes the FASTA headers are named in a common format. Something like "NODE_1_length_263127_cov_73.826513". The first field is a unique name for the contig (just a numerical value), the next field is the length of the sequence, and the last field is the kmer coverage of that contig (this is different than read coverage which NCBI submissions require). Furthermore, the contigs are organized by length where the longest contigs are first.

* Clean up Spades directory.

When you listed the SPAdes directory you could see that it makes a lot of output folders, all of which are explained in the manual. We really only 'care' about the contigs.fasta file and the spades.log. The spades.log is important because it includes details about how exactly we ran the assembly. If you ever wanted to reproduce your assembly this file might come in handy. The rest of the files can be deleted, if we ever need them you can always use the spades.log to rerun the analysis.

We are going to proceed to remove these unwanted files. **Remember if you delete a file using the 'rm' command it is gone forever, there is no way to get it back, there is no recover from trash bin, so be careful!**


```bash
# There are many ways to do this, proceed however you are comfortable. I am going to move the files I want to keep out of the directory, delete everything else in the directory with 'rm' then move my files back in. Alternatively you could just remove unwanted files one at a time.
# Move into the spades directory
cd spades_spades_assembly_default/
# move the files we want to keep back one directory
mv contigs.fasta spades.log ../
# confirm that the files moved!!!!!!!
ls ../
# confirm you are still in the spades_directory (I'm paranoid)
pwd
# you should see something like /home/GROUP/UserName/mdibl-t3-2018-WGS/spades_directory_default/
# After you confirm the files have been moved and you are in the right directory, delete the unwanted files
rm -r *
# move the files back
mv ../contigs.fasta ../spades.log ./
# list the directory and you should see just the two files.
ls
```
