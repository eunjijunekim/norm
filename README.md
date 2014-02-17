## Normalization



### 0. Installation

* git clone

* blast

### 1. Run BLAST
#####A. Create `reads.fa`
Skip this step if the reads were aligned with RUM. 

	perl runall_sam2readsfa.pl <sample dirs> <loc> <sam file name>

> `sam2readsfa.pl` available for running one sample at a time  

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* &lt;sam file name> : the name of sam file (e.g. RUM.sam)

This outputs a file called `reads.fa` of all samples to corresponding sample directories.

#####B. Run BLAST

	perl runall_runblast.pl <sample dirs> <loc> <samfile name> <blast dir> <db>

> `runblast.pl` available for running one sample at a time

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* &lt;samfile name> : the name of sam file 
* &lt;blast dir> : the blast dir (full path)
* &lt;db> : database (full path)

This creates `ribosomalids.txt` and `total_num_reads.txt` of all samples.

#### [Normalization Factor 1: ribo percents]: 
`perl get_ribo_percents.pl > ribo_percents.txt`

### 2. Run Filter
This step removes all rows from input sam file except those that satisfy all of the following:

  1. Unique mapper / Non-Unique mapper
  2. Both forward and reverse map consistently
  3. id not in (the appropriate) file specified in &lt;more ids>
  4. Only on a numbered chromosome, X or Y
  5. Is a forward mapper (script outputs forward mappers only). Will output &lt;target num> read (pairs) (put 0 for this arg if you want to output all).

Run the following command:

    perl runall_filter.pl <sample dirs> <loc> <sam file name> [options]

> `filter_sam.pl` available for running one sample at a time

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path)
* &lt;loc> : the path of the directory with the sample directories
* &lt;sam file name> :  the name of sam file (e.g. RUM.sam)
* option:<br>
  **-u** : set this if you want to return only unique mappers<br>
  **-nu** :  set this if you want to return only non-unique mappers

This creates directories called `Unique` and `NU` in each sample directory and outputs `filtered.sam` files of all samples to the directories created. By default it will return both unique and non-unique mappers. 

### 3. Quantify Exons
##### A. Create Master List of Exons

* Get master list of exons from a UCSC gene info file.

	    perl get_master_list_of_exons_from_geneinfofile.pl <gene info file>

	* &lt;gene info file> : a UCSC gene annotation file including chrom, strand, txStrand, txEnd, exonCount, exonStarts, exonEnds, and name.

 This outputs a file called `master_list_of_exons.txt`.

* Create a study-specific master list of exons by adding novel exons from the study to the `master_list_of_exons.txt` file

		perl make_new_master_list_of_exons.pl <sample dirs> <loc> <master list of exons>

	* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path) 
	* &lt;loc> : the path of the directory with the sample directories
	* &lt;master list of exons> : the `master_list_of_exons.txt` file (with full path)

 This creates a txt file called `NEW_master_list_of_exons.txt`.

##### B. Run quantify exons

This step takes filtered sam files and splits them into 1, 2, 3, 4, 5 exonmappers and notexonmappers. 

Run the following command with **&lt;output sam?> = true**:

	perl runall_quantify_exons.pl <file names> <loc> <exons> <output sam?> [options]

> `quantify_exons.pl` available for running one sample at a time

* &lt;file names> : a file with the names of the `filtered.sam` files (without path)
* &lt;loc> : the path of the directory with the `filtered.sam` files
* &lt;exons> : the `NEW_master_list_of_exons.txt` file (with full path)
* &lt;output sam?> : true
* option:<br>**-NU-only** : set this for non-unique mappers

This outputs multiple files of all samples: `exonmappers.(1, 2, 3, 4, 5).sam`, `notexonmappers.sam`, and `exonquants` file. 

##### C. Downsample

Downsampling is performed for each type of `exonmappers.sam`. Repeat the following steps for all 5 types of exonmappers. (*Example given for 1 exonmapper.*)

* Count the number of reads

		perl wc_all.pl <files> <outfile name>

	* &lt;files> : create a file with the names of the one exonmapper files (e.g. ls *exonmappers.1.sam > files.1.txt)
	* &lt;outfile name> : output file name (e.g. 1_exon_counts.txt)

 This outputs a txt file containing the line counts of all samples and the minimum line count.

* Take the minimum line count from `1_exon_counts.txt` file and generate a sam file with the equal number of lines of all sample
		
		perl runall_head.pl <files> <loc> <num>

	* &lt;files> : a file with the names of the one exonmapper files
	* &lt;loc> : the path to the exonmapper files
	* &lt;num> : minimum line count

 This creates a sam file with the &lt;num> rows of all samples.

* Make sure you perform **C** for all 5 exonmappers before proceeding.

##### D. Concatenate downsampled `exonmappers.(1, 2, 3, 4, 5).sam` files

	perl cat_exonmappers.pl <sample dirs> <loc>

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path) 
* &lt;loc> : the path to the `exonmappers.sam` files

This creates a directory called `normalized_exonmappers` and outputs a sam filed called `norm.sam` of all samples to the directory created.

#### [Normalization Factor 2: exon to nonexon signal]: 
`perl get_exon2nonexon_signal_stats.pl <exonquants files> > exon2nonexon_signal_stats.txt`

> &lt;exonquants files> : a file with the names of the exonquants files

#### [Normalization Factor 3: one exon vs multi exons]: 
`perl get_1exon_vs_multi_exon_stats.pl <exonquants files> > 1exon_vs_multi_exon_stats.txt`

> &lt;exonquants files> : a file with the names of the exonquants files

##### E. Quantify exons for the normalized exonmappers

Run the following command with **&lt;output sam?> = false**:

	perl runall_quantify_exons.pl <file names> <loc> <exons> <output sam?> [options]

> `quantify_exons.pl` available for running one sample at a time


* &lt;file names> : a file with the names of the normalized exonmappers files (without path)
* &lt;loc> : the path of the directory with the normalized exonmappers files
* &lt;exons> : the `NEW_master_list_of_exons.txt` file (with full path)
* &lt;output sam?> : false
* option:<br>**-NU-only** : set this for non-unique mappers

 This outputs a file called `exonquants` of all samples.

**F. Master table of exons counts**

* [Option 1] Get spreadsheet for both Unique and Non-Unique mappers

		perl quants_min_max.pl <file names> <feature type> <loc>

	* &lt;file names> : a file with the names of the exonquants files **sorted by group/condition** (without path)
	* &lt;feature type> : the type of quants file (e.g: exonquants)
	* &lt;loc> : the path of the directory with the Unique and NU directory

 This creates a directory called `quants_all` and outputs a file called `exonquants.MIN_MAX` of all samples to the directory created. The min value is based on the Unique mappers only and the max value is based on all features.

 	  perl quants2spreadsheet_min_max.pl <file names> <type of quants file> <loc>

 	* &lt;file names> : a file with the names of the `exonquants.MIN_MAX` files **sorted by group/condition** (without path)
	* &lt;type of quants file> : the type of quants file (e.g. exonquants)
	* &lt;loc> : the path to `quants_all` directory

 This outputs two spreadsheets called `list_of_exons_counts_MIN.txt` and `list_of_exons_counts_MAX.txt`, containing min and max exon counts of all samples.

* [Option 2] Get spreadsheet for either Unique or Non-Unique mappers

		perl quants2spreadsheet.1.pl <file names> <type of quants file> <loc>

	* &lt;file names> : a file with the names of the exonquants files **sorted by group/condition** (without path)
	* &lt;type of quants file> : the type of quants file (e.g. exonquants)
	* &lt;loc> : the path to the exonquants files 

 This outputs a txt file called `list_of_exons_counts.txt` to `Unique` or `NU` directory, containing exon counts of all samples.

* Annotate

	 	qlogin -l h_vmem=6G
 		perl annotate.pl <annotation file> <features file> > <outfile>

	* &lt;annotation file> : should be downloaded from UCSC known-gene track including at minimum name, chrom, strand, exonStarts, exonEnds, all kgXref fields and hgnc, spDisease, protein and gene fields from the Linked Tables table.
	* &lt;features file> : the `list_of_exons_counts.txt`, `list_of_exons_counts_MIN.txt`, or `list_of_exons_counts_MAX.txt` file
	* &lt;outfile> : output file name (e.g. `master_list_of_exons_counts.txt`, `master_list_of_exons_counts_MIN.txt` or `master_list_of_exons_counts_MAX.txt`)

 This adds annotation to the list of exons counts file.

### 4. Quantify Introns
##### A. Create Master List of Introns

    perl get_master_list_of_introns_from_geneinfofile.pl <gene info file>

* &lt;gene info file> : a UCSC gene annotation file including chrom, strand, txStrand, txEnd, exonCount, exonStarts, exonEnds, and name.

This outputs a txt file called `master_list_of_introns.txt`.

##### B. Run quantify introns

This step takes `notexonmappers.sam` files and splits them into 1, 2, 3, 4, 5, 6, 7, 8 intronmappers and intergenicmappers files. 

Run the following command with **&lt;output sam?> = true**:

	perl runall_quantify_introns.pl <file names> <loc> <introns> <output sam?>

> `quantify_introns.pl` available for running one sample at a time

* &lt;file names> : a file with the names of the `notexonmappers.sam` files (without path)
* &lt;loc> : the path of the directory with the `notexonmappers.sam` files
* &lt;introns> : the `master_list_of_introns.txt` file (with full path)
* &lt;output sam?> : true

This outputs multiple files of all samples: `intronmappers.(1, 2, 3, 4, 5, 6, 7, 8).sam`, `intergenicmappers.sam`, and `intronquants` file.

##### C. Downsample

Downsampling is performed for each type of `intronmappers.sam` and the `intergenicmappers.sam` file. Repeat the following steps for all 8 types of intronmappers and the intergenicmappers. (*Example given for 1 intronmapper.*)

* Count the number of reads

		perl wc_all.pl <files> <outfile name>

	* &lt;files> : a file with the names of the one intronmapper files (e.g. ls *intronmappers.1.sam > int.files.1.txt)
	* &lt;outfile name> : output file name (e.g. 1_intron_counts.txt)

 This outputs a txt file containing the line counts of all samples and the minimum line count.

* Take the minimum line count from `1_intron_counts.txt` file and generate a sam file with the equal number of lines of all samples
		
		perl runall_head.pl <files> <loc> <num>

	* &lt;files> : a file with the names of the one intronmapper files
	* &lt;loc> : the path to the intronmapper files
	* &lt;num> : minimum line count

 This creates a sam file with the &lt;num> rows of all samples.

* Make sure you perform **C** for all 8 intronmappers and intergenicmappers before proceeding.

#####D. Concatenate downsampled `intronmappers.(1, 2, 3, 4, 5, 6, 7, 8).sam` files

	perl cat_intronmappers.pl <sample dirs> <loc>

* &lt;sample dirs> : a file with the names of the sample directories with SAM file/alignment output (without path) 
* &lt;loc> : the path to the intronmappers files

This creates a directory called `normalized_intronmappers` and outputs a file called `norm.sam` of all samples to the directory created. It also creates `normalized_intergenicmappers` directory and puts the normalized intergenicmappers files to the directory.

##### E. Quantify introns for the normalized intronmappers

Run the following command with **&lt;output sam?> = false**:

	perl runall_quantify_introns.pl <file names> <loc> <introns> <output sam?>

> `quantify_introns.pl` avilable for running one sample at a time

* &lt;file names> : a file with the names of the normalized intronmappers files (without path)
* &lt;loc> : the path of the directory with the normalized intronmappers files
* &lt;introns> : the `master_list_of_intron.txt` file (with full path)
* &lt;output sam?> : false

This outputs a file called `intronquants` of all samples.

##### F. Master table of introns counts

* [Option 1] Get spreadsheet for both Unique and Non-Unique mappers

		perl quants_min_max.pl <file names> <feature type> <loc>

	* &lt;file names> : a file with the names of the intronquants files **sorted by group/condition** (without path)
	* &lt;feature type> : the type of quants file (e.g: intronquants)
	* &lt;loc> : the path of the directory with the Unique and NU directory

 This creates a directory called `quants_all` and outputs a file called `intronquants.MIN_MAX` of all samples to the directory created. The min value is based on the Unique mappers only and the max value is based on all features.

	  perl quants2spreadsheet_min_max.pl <file names> <type of quants file> <loc>

 	* &lt;file names> : a file with the names of the `intronquants.MIN_MAX` files **sorted by group/condition** (without path)
	* &lt;type of quants file> : the type of quants file. (e.g. intronquants)
	* &lt;loc> : the path to the `quants_all` directory

 This outputs two spreadsheets called `master_list_of_introns_counts_MIN.txt` and `master_list_of_introns_counts_MAX.txt`, containing min and max intron counts of all samples.

* [Option 2] Get spreadsheet for either Unique or Non-Unique mappers

		perl quants2spreadsheet.1.pl <file names> <type of quants file> <loc>

	* &lt;file names> : a file with the names of the intronquants files **sorted by group/condition** (without path)
	* &lt;type of quants file> : the type of quants file (e.g. intronquants)
	* &lt;loc> : the path to the intronquants files 

 This outputs a txt file called `master_list_of_exons_counts.txt`, containing intron counts of all samples.

### 5. Quantify Junctions

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



