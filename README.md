# CRISPR_Screen
## CRISPRi/a/ko Screen Data Generation Tips

Most sgRNA libraries are housed in the LentiCRISPRv1 or [v2]([url](https://www.addgene.org/52961/)) backbones. 

i.	Guides are inserted into typeIIS restriction sites (see below) downstream of U6

ii.	Transduce with pool, proceed with positive or negative selection based on desired phenotype, and isolate gDNA from your filtered population. Pre-selection cells are used as a control. 

iii.	Libraries are made in a 1- or 2-step PCR reaction priming the guide-containing region from the lenti insert. See below for priming sites. Template + product ≈ 350bp

![image](https://user-images.githubusercontent.com/44478133/212573868-dc488b3d-790a-4458-91b7-9bd8e0139a67.png)

iv.	Primers contain the whole illumina adapter needed for sequencing. 

- The P5 primer (fwd) is usually not barcoded. It’s also ordered as a pool, staggered by 1nt for library complexity. Read 1 uses this primer.
  
- The P7 primer (rev) contains the barcode (index 1). 

![image](https://user-images.githubusercontent.com/44478133/212574320-24e673d2-4d31-4aec-940e-e88047356da3.png)

v.	I recommend using a Nextseq 500/550 75-cycle high-output kit (or NextSeq 2000 P3 50 cycle kit) for sequencing multiple libraries. Minimum coverage should == #guides x 200. Allocate cycles as 75 | 8 | 0 | 0 (R1 | I1 |I2 |R2). You’re only sequencing from P5 on the U6 end. 

vi.	Run sequencer in Manual mode, opting for monitoring AND storage on basespace. Upload a sample sheet saved as CSV. Sheet below corresponds to primers above. Experiment name should be unique—you’ll need this to download your FASTQs

![image](https://user-images.githubusercontent.com/44478133/212574367-89777e3d-237a-4fda-992b-55169fa2c355.png)

## CRISPR Library Data Analysis

Consult the relevant publicatioin before processing to see whether your libraries are compatible with [MAGeCK]([url](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-014-0554-4)). It should work with all the lenti backbones. It has a published [protocol]([url](https://www.nature.com/articles/s41596-018-0113-7)), but I don’t find it particularly helpful. 

1.	You need a 3-column sgRNA library table. ID should be unique. Check the supplementary information of the article describing your library. They should have a table like this. Get these three columns, name the columns, save as “library.csv” and upload to the cluster.

![image](https://user-images.githubusercontent.com/44478133/212574040-19e90867-0d9f-4f75-8d83-acb72a6faa38.png)

2.	Log in to the cluster and start an interactive job:

```
#Here is a job with 15 cores, 30G RAM

bsub -n 15 -R "span[hosts=1]" -R "rusage[mem=2048]" -W 4:00 -q interactive -Is bash
```

3.	Get your sequencing data onto the cluster. I recommend the illumina commandline uitility. Follow [setup instructions]([url](https://developer.basespace.illumina.com/docs/content/documentation/cli/cli-overview)) if first time. Find your project and download.

```
#Find your project and id. It’s the experiment name above
bin/bs list projects
```

![image](https://user-images.githubusercontent.com/44478133/212574088-37b4e40e-2c9e-455b-a1ac-69e387eb99d1.png)

```
# Download and name output folder
bin/bs download projects -i 378654281 -o CRISPRScreen_output
```

4.	Your files should be named something like “Sample01_R1_L001_001.fastq.gz”. If you used a Nextseq, you’ll need to combine files from all 4 lanes before proceeding. This isn’t an issue on 1-lane machines like the Miniseq, or FASTQs from outside vendors. 

```
cd CRISPRScreen_output
#Schedule jobs to merge the lanes (all one line)
for i in *L001_*gz; do bsub -n 4 -R "span[hosts=1]" -R "rusage[mem=1024]" -o myjob.out -e myjob.err -oo myjob.out -eo myjob.err -W 3:59 -q short -J "$i" "zcat ${i%_L00*}*.gz > ${i%_S*}_R1.fq && gzip ${i%_S*}_R1.fq";done
#Remove input FASTQs
rm *L00*.gz
```

5.	MAGeCK isn’t on the cluster as a module, so I made a docker image compatible with singularity. Enter the image and start the software. First time will take a couple minutes to download the container from docker and convert to singularity image. 

```
singularity shell docker://umasstr/mageckvispr:latest
source activate mageck-vispr
```

6.	Align and count sgRNAs for all samples. Make sure you have your library file from step 1.

```
# mageck count -l <library csv file> -n <name of analysis> --sample-label <comma separated sample names>  --fastq <space separated FASTQs>
mageck count -l library.csv -n Screen --sample-label Sample1,Sample2,Sample3,Sample4,Sample5 --fastq Sample1_R1.fq.gz Sample2_R1.fq.gz Sample3_R1.fq.gz Sample4_R1.fq.gz Sample5_R1.fq.gz
#Your output is:
# Screen.count.txt
# Screen.count_normalized.txt
# Screen.count_report.Rmd
# Screen.countsummary.txt
# Screen.log
# Screen_countsummary.R
# Screen_countsummary.Rnw
```

7.	Now, run the comparisons between your experimental and control samples. Here, Sample01 is experimental and Sample02 is a control. 

```
# mageck test -k <count table> -t <EXP> -c <CTRL> -n <name output>
mageck test -k Screen.count.txt -t Sample01 -c Sample02 -n 01vs02
```

8.	The output file “01vs02.gene_summary.txt”  contains enrichment information for your guides. They’re ranked by positive and negative selection:

![image](https://user-images.githubusercontent.com/44478133/212574169-63d52a0c-1fc3-4c93-88f5-c4cce4f9d10c.png)
