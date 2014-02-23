## Normalization

### 0. Setting Up

#####A. Download


#####B. Install BLAST
Download [BLAST and BLAST databases] (http://blast.ncbi.nlm.nih.gov/Blast.cgi?PAGE_TYPE=BlastDocs&DOC_TYPE=Download)

#####C. Input Directory Structure
Make sure your alignment outputs(sam files) are in each sample directory inside the `Aligned_DATA` folder.
<pre>
STUDY					
└── Aligned_DATA					
    ├── Sample_1					
    │   └── Aligned.sam					
    ├── Sample_2					
    │   └── Aligned.sam					
    ├── Sample_3					
    │   └── Aligned.sam					
    └── Sample_4					
        └── Aligned.sam					
</pre>					

#####D. Output Directory Structure
Once you complete the normalization pipeline, your directory structure will look like this:
<pre>
STUDY
│── Aligned_DATA
│   ├── Sample_1
│   │   ├── NU
│   │   └── Unique
│   ├── Sample_2
│   │   ├── NU
│   │   └── Unique
│   ├── Sample_3
│   │   ├── NU
│   │   └── Unique
│   └── Sample_4
│       ├── NU
│       └── Unique
│
└── NORMALIZED_DATA
    ├── exonmappers
    │   ├── MERGED
    │   ├── NU
    │   └── Unique
    ├── notexonmappers
    │    ├── MERGED
    │    ├── NU
    │    └── Unique
    ├── FINAL_SAM
    │   ├── MERGED
    │   ├── NU
    │   └── Unique
    └── Junctions
</pre>
    					
### 1. Run BLAST

	perl runall_runblast.pl <sample dirs> <loc> <samfile name> <blast dir> <db>

> `runblast.pl` available for running one sample at a time

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* &lt;samfile name> : the name of sam file (e.g. RUM.sam, Aligned.out.sam)
* &lt;blast dir> : the blast dir (full path)
* &lt;db> : database (full path)

This outputs `ribosomalids.txt` and `total_num_reads.txt` of all samples.

### 2. Run Filter
This step removes all rows from input sam file except those that satisfy all of the following:

  1. Unique mapper / Non-Unique mapper
  2. Both forward and reverse map consistently
  3. id not in (the appropriate) file specified in &lt;more ids>
  4. Only on a numbered chromosome, X or Y
  5. Is a forward mapper (script outputs forward mappers only). Will output &lt;target num> read (pairs) (put 0 for this arg if you want to output all).

Run the following command. By default it will return both unique and non-unique mappers.

    perl runall_filter.pl <sample dirs> <loc> <sam file name> [options]

> `filter_sam.pl` available for running one sample at a time

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* &lt;sam file name> :  the name of sam file
* option:<br>
  **-u** : set this if you want to return only unique mappers<br>
  **-nu** :  set this if you want to return only non-unique mappers

This creates directories called `Unique` and `NU` in each sample directory and outputs `filtered.sam` files of all samples to the directories created. 

### 3. Quantify Exons
##### A. Create Master List of Exons
Get master list of exons from a UCSC gene info file.

	perl get_master_list_of_exons_from_geneinfofile.pl <gene info file>

* &lt;gene info file> : a UCSC gene annotation file including chrom, strand, txStrand, txEnd, exonCount, exonStarts, exonEnds, and name.

This outputs a file called `master_list_of_exons.txt`.

##### B. Run quantify exons

This step takes filtered sam files and splits them into 1, 2, 3 ... 20 exonmappers and notexonmappers. 

Run the following command with **&lt;output sam?> = true**. By default this will return the Unique exonmappers. Use -NU-only to get Non-Unique exonmappers:

	perl runall_quantify_exons.pl <sample dirs> <loc> <exons> <output sam?> [options]

> `quantify_exons.pl` available for running one sample at a time

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* &lt;exons> : the `master_list_of_exons.txt` file (with full path)
* &lt;output sam?> : true
* option:<br>**-NU-only** : set this for non-unique mappers

This outputs multiple files of all samples: `exonmappers.(1, 2, 3, 4, ... 20).sam`, `notexonmappers.sam`, and `exonquants` file to `Unique` / `NU` directory inside each sample directory. 

##### C. Normalization Factors
* Ribo percents: 

		perl runall_get_ribo_percents.pl <sample dirs> <loc>

	* &lt;sample dirs> : a file with the names of the sample directories
	* &lt;loc> : the location where the sample directories are

 It assumes there are files of ribosomal ids output from runblast.pl
each with suffix "ribosomalids.txt". This will output `ribosomal_counts.txt` and `ribo_percents.txt`.

* Exon to nonexon signal:

		perl get_exon2nonexon_signal_stats.pl <sample dirs> <loc>

	* &lt;sample dirs> : a file with the names of the sample directories
	* &lt;loc> : the location where the sample directories are
	* option:<br>
  	**-u** : set this if you want to return only unique stats, otherwise by default it will return both unique and non-uniqe stats<br>
  	**-nu** :  set this if you want to return only non-unique statsotherwise by default it will return both unique and non-uniqe stats

 This will output `exon2nonexon_signal_stats_Unique.txt` and/or `exon2nonexon_signal_stats_NU.txt` depending on the option provided.

* One exon vs multi exons:
	
		perl get_1exon_vs_multi_exon_stats.pl  <sample dirs> <loc>

	* &lt;sample dirs> : a file with the names of the sample directories
	* &lt;loc> : the location where the sample directories are
	* option:<br>
  	**-u** : set this if you want to return only unique stats, otherwise by default it will return both unique and non-uniqe stats<br>
	**-nu** :  set this if you want to return only non-unique statsotherwise by default it will return both unique and non-uniqe stats

 This will output `1exon_vs_multi_exon_stats_Unique.txt` and/or `1exon_vs_multi_exon_stats_NU.txt` depending on the option provided.

### 4. Quantify Introns
##### A. Create Master List of Introns

    perl get_master_list_of_introns_from_geneinfofile.pl <gene info file>

* &lt;gene info file> : a UCSC gene annotation file including chrom, strand, txStrand, txEnd, exonCount, exonStarts, exonEnds, and name.

This outputs a txt file called `master_list_of_introns.txt`.

##### B. Run quantify introns

This step takes `notexonmappers.sam` files and splits them into 1, 2, 3 ... 10 intronmappers and intergenicmappers files. 

Run the following command with **&lt;output sam?> = true**. By default this will return the Unique intronmappers. Use -NU-only to get Non-Unique intronmappers:

	perl runall_quantify_introns.pl <sample dirs> <loc> <introns> <output sam?> [options]

> `quantify_introns.pl` available for running one sample at a time

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* &lt;introns> : the `master_list_of_introns.txt` file (with full path)
* &lt;output sam?> : true
* option:<br>**-NU-only** : set this for non-unique mappers

This outputs multiple files of all samples: `intronmappers.(1, 2, 3, ... 10).sam`, `intergenicmappers.sam`, and `intronquants` file.

### 5. Downsample

##### A. Run head 
	
	perl runall_head.pl <sample dirs> <loc>

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories

This will output the same number of rows from each file in each Sample Unique and NU directory of the same type.

##### B. Concatenate head files

	perl cat_headfiles.pl <sample dirs> <loc> [options]

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* option:<br>
  -u  :  set this if you want to return only unique mappers, otherwise by default
         it will return both unique and non-unique mappers.<br>
  -nu :  set this if you want to return only non-unique mappers, otherwise by default
         it will return both unique and non-unique mappers.

This will create `NORMALIZED_DATA`, `NORMALIZED_DATA/exonmappers`, and `NORMALIZED_DATA/notexonmappers` directories and output normalized exonmappers, intronmappers and intergenic mappers of all samples to the directories created.

##### C. Merge normalized SAM files

	perl make_final_samfile.pl <sample dirs> <loc> [options]

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* option:<br>
  -u  :  set this if you want to return only unique mappers, otherwise by default
         it will return both unique, non-unique, and merged final sam files.<br>
  -nu :  set this if you want to return only non-unique mappers, otherwise by default
         it will return both unique, non-unique, and merged final sam files.

This will create `FINAL_SAM`. Then, depending on the option given, it will make `FINAL_SAM/Unique`, `FINAL_SAM/NU`, and/or `FINAL_SAM/MERGED` directory and output final sam files to the directories created.

### 6. Quantify Junctions



* [Option 1] Use both Unique and Non-Unique mappers

 **A. Concatenate normalized Unique and Non-Unique exonmappers**
 
	  perl cat_exonmappers_Unique_NU.pl <files> <loc>
 
 * &lt;files> : a file with the names of the normalized exonmappers (without path)
 * &lt;loc> : the path of the directory with the Unique and NU directory
 
 This creates a directory called `junctions_all` and outputs a file called `exonmappers.ALL.norm.sam` of all samples to the directory created.
 
 **B. Run sam2junctions**
 
      perl runall_sam2junctions.pl <file names> <loc> <genes> <genome>
 
 * &lt;file names> : a file with the names of the `exonmappers.ALL.norm.sam` files (without path)
 * &lt;loc> : the path to the dir with the `exonmappers.ALL.norm.sam` files
 * &lt;genes> :the RUM gene info file (with full path)
 * &lt;genome> : the RUM genome sequene one-line fasta file (with full path)
 
 This outputs `junctions_hq.bed`, `junctions_all.bed` and `junctions_all.rum` files for each  sample.
 
 **C. Master table of junctions counts**
 
	  perl juncs2spreadsheet_min_max.pl <file names> <loc>
 
 * &lt;file names> : a file with the names of the `junctions_all.rum` file **sorted by group/condition** (without path)
 * &lt;loc> : the path to the junctions_all directory
 
 This outputs `master_list_of_junctions_counts_MIN.txt` and `master_list_of_junctions_counts_MAX .txt` file, where MIN score is long overlap unique reads and max score is long overlap unique reads + long overlap NU reads.

* [Option 2] Use Unique mappers Only

 **A. Run sam2junctions**
 
      perl runall_sam2junctions.pl <file names> <loc> <genes> <genome>
 
 * &lt;file names> : a file with the names of the `exonmappers.norm.sam` files (without path)
 * &lt;loc> : the path to the dir with the files
 * &lt;genes> : the RUM gene info file (with full path)
 * &lt;genome> : the RUM genome sequene one-line fasta file (with full path)
 
 This outputs `junctions_hq.bed`, `junctions_all.bed` and `junctions_all.rum` files for each  sample.

 **B. Master table of junctions counts**
 
	  perl juncs2spreadsheet.1.pl <file names> <loc>
 
 * &lt;file names> : a file with the names of the `junctions_all.rum` file **sorted by group/condition** (without path)
 * &lt;loc> : the path to the `junctions_all.rum` files
 
 This outputs a file called `master_list_of_junctions_counts.txt` file to the `Unique` directory.

### 6. Merge normalized sam and quants files

* Create a normalized sam file (final ver) of all samples by merging normalized exon, intron, and intergenicmappers

	 perl cat_normalized_samfiles.pl <sample dirs> <loc>

	* &lt;sample dirs> : a file with the names of the sample directories with RUM output (without path)
	* &lt;loc> : the path of the directory with the normalized_exonmappers, normalized_intronmappers and normalized_intergenicmappers directory

 This creates a directory called `FINAL_SAM` and outputs `FINAL.norm.sam` file of all samples to the directory. 

* Label, concatenate exons, junctions, and introns counts and filter low expressors

	* Label the master list of counts files

 	 		perl label_quants_file.pl <files> <loc>

 		* &lt;files> : a file with the names of the master list of counts files (without path)
		* &lt;loc> is the path to the master list of counts files

	 This outputs labeled master list of counts files.

	* Concatenate exon, intron, and junctions counts files

		 	perl cat_master_list_of_counts.pl <files> <loc>

		* &lt;files> : a file with the names of the master list of counts files
		* &lt;loc> : the path of the directory with the labeled master list of counts files

	 This creates a merged master list of counts file.
	
	* Filter low expressors

			perl filter_low_expressors.pl <file> <number_of_samples> <cutoff> > <outfile name>

		* &lt;file> : the merged master list of counts file without path
		* &lt;number_of_samples> : number of samples
		* &lt;cutoff> : cutoff value
		* &lt;outfile name> : outfile name (e.g. `master_list_of_counts_FINAL.txt`)

 	 This creates the FINAL spreadsheet. 



