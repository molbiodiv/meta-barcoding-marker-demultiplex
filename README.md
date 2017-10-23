# metabc-marker-demultiplex

## What it does

Demultiplexing of metabarcoding data which consists of multiple markers.
* Data must have followed a library preparation/sequencing strategy which includes sequencing of the forward primers.
* Data must be demultiplexed for samples already
* Sequence data must be in forward orientation

The categorization is based on Hidden Markov Model (HHMs) hits of the forward primer within the first 20 bp. This is very fast, and allows high throughput of the data.

## Citation

If you use this script and the HMMs, please cite our paper where we established this strategy. It's on it's way to getting submitted, updated info will be put here once it is published.

## Requirements

* **SeqFilter:**

```sh
git clone https://github.com/BioInf-Wuerzburg/SeqFilter
cd SeqFilter
make
cd  ..

```
* **Usearch:** A copy of Usearch >=9.x freely available here after registration, 32bit is sufficient: https://www.drive5.com/usearch/manual/

* **HMMer:** get a free copy here and install: http://hmmer.org

## Predefined HMMs

### Plants:

* Rbcl (long version with ~ 700 bp): hmm_rbcl.hmm
 * forward primer: *CTTACCAGCCTTGATCGTTA*
* Rbcl (short version with ~ 500 bp): hmm_rbcl_pollen.hmm
 * forward primer: *ATGTCACCACAAACAGAGACTAAAGC*
* psbA-trnH: hmm_psba.hmm
 * forward primer: *GTTATGCATGAACGTAATGCTC*
* ITS2: hmm_its2.hmm
 * forward primer: *ATGCGATACTTGGTGTGAAT*

### Animals:

* COI: hmm_coi.hmm
 * forward primer: *GGWACWGGWTGAACWGTWTAYCCYCC*

## Train HMMs yourself for your primers

```sh
hmmbuild hmm_marker.hmm primers_marker.fasta

```

## Separation

Preparations:

```sh

data=$(pwd) # set data directory, in this case here the local one we are in:
s=./SeqFilter/bin/SeqFilter # location of Seqfilter
u=./usearch9.2.64_i86osx32 # location of USearch
hmm=./hmms # location of HMMs

# in case you files are gzipped (like from the MiSeq)
gunzip *.gz

# how are the forward and reverse reads labelled (here MiSeq defaults)
RF='_R1_001.fastq';
RR='_R2_001.fastq';
```

Things done in the following:
* Speeding things up, only 20 bp for speedup, we just want to get the IDs anyway
* Translation into fasta for HMMer to work
* Hold these against the hmms and see which primers it belongs to
* store the IDs
* filter them with Seqfilter in marker specific fasta files.

** This example: rbcl, its2 and psba-trnH. ** You may need to adapt to your purpose.

```sh

for file in `cat samples.txt` ;
do
  $u -fastq_filter $data/$file$RF -fastaout $data/20bp_$file.fasta -fastq_trunclen 20
  hmmsearch --tblout hits_rbcl_$file.txt  $hmm/hmm_rbcl_pollen.hmm $data/20bp_$file.fasta
  hmmsearch --tblout hits_psba_$file.txt  $hmm/hmm_psba.hmm $data/20bp_$file.fasta
  hmmsearch --tblout hits_coi_$file.txt  $hmm/hmm_coi.hmm $data/20bp_$file.fasta

  cat hits_rbcl_$file.txt | grep -v "^#" | cut -f 1 -d" " > hitsHeader_rbcl_$file.txt
  cat hits_psba_$file.txt | grep -v "^#" | cut -f 1 -d" " > hitsHeader_psba_$file.txt
  cat hits_coi_$file.txt | grep -v "^#" | cut -f 1 -d" " > hitsHeader_coi_$file.txt
  cat hitsHeader_rbcl_$file.txt hitsHeader_psba_$file.txt hitsHeader_coi_$file.txt  > hitsHeader_xxxx_$file.txt

  $s --ids hitsHeader_rbcl_$file.txt $data/$file$RF -o $data/rbcl_$file$RF
  $s --ids hitsHeader_rbcl_$file.txt $data/$file$RR -o $data/rbcl_$file$RR

  $s --ids hitsHeader_psba_$file.txt $data/$file$RF -o $data/psba_$file$RF
  $s --ids hitsHeader_psba_$file.txt $data/$file$RR -o $data/psba_$file$RR

  $s --ids hitsHeader_coi_$file.txt $data/$file$RF -o $data/coi_$file$RF
  $s --ids hitsHeader_coi_$file.txt $data/$file$RR -o $data/coi_$file$RR

  $s --ids hitsHeader_xxxx_$file.txt $data/$file$RF -o $data/xxxx_$file$RF --ids-exclude
  $s --ids hitsHeader_xxxx_$file.txt $data/$file$RR -o $data/xxxx_$file$RR --ids-exclude

done
```

Cleaning up what we produced, a lot of temp files can be removed.

```sh

rm hits*
rm 20bp*

```

Done, you now have fastq files separated by marker, starting with a corresponding prefix.
