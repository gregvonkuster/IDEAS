—————————————————————————————————————————————————
	IDEAS v1.0 for Linux, by Yu Zhang	
—————————————————————————————————————————————————

1. INSTALL: 
	Just unzip and untar everything to a folder.


2. RUN:
	./ideas inputfile [options]
	
	Options:

//////////////(A) parameter controls///////////////////////

	-log2 		Log2-transformation of the input data by log2(x+1), recommended for read count data to reduce skewness.
			Alternatively, you can use -log2 num, then the program will take log2(x+num) transformation.

	-o output	Specify the prefix of output file; default is output prefix=inputfile.

	-G num		Specify the maximum number of states to be inferred. 
			Use this option if you want to constrain the maximum number of states to be generated by IDEAS; the final number of inferred states may be smaller than the number you specified.
	
	-C num		Specify the initial number of states; default is 20.
			Use this option if the number of states you expect to generate is greater than 20. Say, if you expect to have 30 states, while IDEAS may infer 30 states or more by starting from just 20 states, it may not do so if it is trapped in a local mode. 
			We recommend setting the initial number of states slightly larger than the number of states you expect or are willing to handle.
	
	-P num		Maximum number of position classes to be inferred. 
			You probably don’t need this option unless in the following two cases:
			a) you do not want position classes (for testing purpose), then use -P 1.
			b) the program runs slow because there are too many position classes (you can see it from screen output or tmp.out file). Usually <100 position classes won’t slow down the program.
	
	-K num		Maximum number of cell type clusters allowed.
			As above, you probably don’t need this option unless for testing purpose. If you set -K 1, then all cell types will be clustered in one group. The program will not do it at the beginning, but eventually the number of cell type clusters will be bounded by this number.

	-A num		Specify the prior concentration parameter; default is A=sqrt(number of cell types).
			A smaller concentration parameter (say, 1 or less) will emphasize more on position specificity, and a larger concentration parameter (say, 10 * number of cell types) will emphasize more on global homogeneity.

	
	-sample burnin_num mcmc_num	Specify the number of burnin and maximization steps; default it is 50 50.
			Increasing these two numbers will increase computing and only slightly increase accuracy.
			Decreasing these two numbers will reduce computing but may also reduce accuracy. 
			We recommend to run IDEAS with at least 20 burnins and 20 maximizations.
			The program will not stop even if it reaches a maximum mode.

	-minerr num	Specify the minimum standard deviation for the emission Gaussian distribution; default is 0.5. 
			However, you may change the default minerr value, especially if the standard deviation of your data is much smaller or much larger than 1.
			For instance, when you run IDEAS, the first line outputs “ysd=xxx”, which is the total standard deviation of your data. If that value is <0.5, you may set minerr to an even smaller number, say xxx/2. If that value is much greater than 1, say 20, you may set minerr to a larger value, say 5. Modifying minerr in the former case is more necessary than in the latter case, because otherwise you may end up finding no interesting segmentations.
			We do not recommend setting minerr to be 0 or smaller, as doing so may capture some artificial and uninteresting states due to tightly clustered data, such as 0 in read counts.
	
	-maxerr num	Specify the maximum standard deviation for the emission Gaussian distribution; default is infinity.
			This is on the contrary to minerr, if you want to find fine-grained states, you may use this option. However, we rarely see use of this unless you need more states to be inferred. 
	


//////////////(B) Incorporating information from other results///////////////////////	

	-startpara parafile 	This option allows you to specify the starting parameter values for the emission Gaussian distribution for states. The parafile is the .para file output by IDEAS.

	-prevrun statefile clusterfile		This option allows you to restore segmentations for the same data from a previous run of IDEAS, and then continue segmentation. However, hyper parameters are not restored from previous runs, and thus this is not an exact continuation.

	
	-otherpara parafile 	This allows you to use .para file from other data as priors for the Gaussian distribution parameters in the current job.
				Using this option, you can borrow data distribution information from other cell types. However, currently we assume that the set of marks and their orders in input are the same between previous and current data.
	
	-otherstate statefile clusterfile	This allows you to use .state and .cluster files from other data as priors for the segmentations in the current job. 
				Using this option, you can borrow segmentation information from other cell types. However, currently we assume that the set of marks and their orders in input are the same between previous and current data.

	-norm		Let IDEAS to standardize the means and variances of all marks in the input data. 
			However, we recommend you do the normalization by yourself, so you know what is exactly being analyzed by IDEAS.



//////////////(C) Parallelization///////////////////////	

	-thread num	Specify the total number of threads to be used for parallelization. This number should be the same as the total number of threads you request from queue.
			See the included myrun.script for an example PBS script to be submitted in queue for running IDEAS in parallel using 64 threads.
	
	
[Disclaimer:]
Running options in (B) and (C) have not been extensively tested, so please report any issues to me and I’ll fix them. Combinations of these options have also not been extensively tested.
	Current version supports missing marks in some cell types, but that function has not been evaluated.

[Additional note:]
This program outputs a tmp.out file that contains running details; you don’t need to read it unless your job is running slowly in the queue and you want to know which iteration it is at.




3. INPUT FORMAT:

	1st line: column names
	remaining lines: position information and data

	An example for 3 marks in 2 cell lines:
	
		chr pos cell1.mark1 cell1.mark2 cell1.mark3 cell2.mark1 cell2.mark2 cell2.mark3
		1 1000200 1 5 0 4 0 12
		1 1000400 4 17 1 1 1 0
		1 1000600 10 31 7 8 2 0
	
	In this example, the first two columns are genome positions (left, center or right end of a window? doesn’t matter, but be consistent for all windows), the next 6 columns are read count data.
	The column names indicate that cell 1 data are provided first, and then cell 2 data. 
		"cell1" etc can be replaced by actual cell type names without space.
		"mark1" etc can also be replaced by actual mark names without space.
		both cell type and mark names need to be provided in the 1st line, and they need to be connected by "." (so DO NOT use ‘.’ in either cell names or mark names).
	The toy.data has identical column names for cell-mark combinations, which indicate replicate data.
	
	If certain mark is missing in a cell type, it is fine, such as:
		
		chr pos cell1.mark1 cell1.mark3 cell2.mark1 cell2.mark2
		1 1000200 1 0 4 0 
		1 1000400 4 1 1 1
		1 1000600 10 7 8 2 
		
	In this example, mark2 in cell1 and mark3 in cell2 are missing.
	However, in such case, the program won’t know the order of marks as you want them to be (e.g., mark1 mark2 mark3). 
	Instead, the order of marks taken by the program will be determined by their order of appearance, and in this example, mark1, mark3, mark2.
	This matters because when reading the output from the program, you want to know which column correspond to which mark in the mean vector for each epigenetic state. Fortunately, I do have the mark names output in the parameter file.

	[Additional notes:]
	
	If you want to specify a window instead of a position for the window, you can do so by inserting a 3rd column in input file as follows
	
		chr pos_st pos_ed cell1.mark1 cell1.mark2 cell1.mark3 cell2.mark1 cell2.mark2 cell2.mark3
		1 1000200 1000400 1 5 0 4 0 12
		1 1000400 1000600 4 17 1 1 1 0
		...
	
	
	This is the same example as above, but with a 3rd column denoting the end of window positions



4. OUTPUT FILES:

	*.state file: 	Epigenetic states and position classes
			First 4 columns are index, chr, position_st position_ed (position_ed will be the same as position_st if only one position for each window is provided in input) 
			The next N columns are epigenetic states, where N=total number of cell types, including replicates. 
			For the toy.data, N=10 because each of 5 cell types has 2 replicates.
			The last column is the position class label.
			All labels starts from 0 (instead of 1).

	*.para:		Frequency, mean and variance parameters for each epigenetic states. 
			Read by readPara() function in plot.R

	*.cluster file: Local cell type clustering result, one row for each replicate of cell type. 
			Shown by showPop() function in plot.R

	*impute* files: if you have missing marks in input, then those will be imputed and output here.
	



5. VISUALIZATION:

	The included plot.R is a R script that visualizes the segmentation results.
	In R environment, load this file by source("plot.R").
	
	(A) Get parameters for epigenetic states: readPara(), 
		Example: 	readPara("toy.data",1,3)
				First argument is input file name.
				Second argument is always 1 as of now.
				Third argument is the number of marks per cell type. If there are missing marks, this number should be the number of marks as if there are no missing marks.
			
				This function outputs an object with 3 components:
				$p: proportion of epigenetic states in the entire data,
				$m: mean vector for all epigenetic states, every k values correspond to one state, where k is the number of marks you specified,
				$v: covariance matrices for all epigenetic states, every k rows correspond to one state.

	(B) Show cell type clustering and epigenetic states: showPop()
		Example:	showPop("toy.data", indlist=1:10, inv=1:1000, repn=2)
				First argument is the inputfile name.
				Second argument list which (replicate of) cell types (their indices, starting from 1) should be shown, if not specified, all will be shown.
				Third argument list which positions (their indices, starting from 1) should be shown, if not specified, all will be shown.
				Fourth argument specifies number of replicates per cell type (only works if the number of replicates for each cell is the same); not needed if no replicates.

				This function will show a figure with three parts, top one is the cell type clustering figure using results in *.cluster file; 
				middle one is the epigenetic state using results in *.state file;
				and bottom one is the position class using results in *.state file.

	
6. REFERENCES

	Yu Zhang, Feng Yue, Ross C. Hardison. Bayesian Modeling of Epigenetic Variation in Multiple Human Cell Types. bioRxiv, doi: http://dx.doi.org/10.1101/018028

__________________________________________________________
Questions and comments?

please contact me at yzz2 at psu.edu , Thanks,
__________________________________________________________


