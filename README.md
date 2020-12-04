# Table of contents

- [DeepTools bamCoverage normalization](#deeptools-bamcoverage-normalization)
- [Relevant files and links](#relevant-files-and-links)
- [How bamCoverage works](#how-bamcoverage-works)
- [Summary](#summary)
- [How to correctly normalize your data 1](#how-to-correctly-normalize-your-data-1)
- [How to correctly normalize your data 2](#how-to-correctly-normalize-your-data-2)

# Introduction 

DeepTools bamCoverage by default does not normalize for library sequencing depth. Since bamCoverage outputs bigwig files and is the pre-requisite function used for generating heatmaps via computeMatrix and plotHeatmap, failing to account for normalization will lead to misleading IGV tracks and figures. For example, let's say library A has 24 M reads whereas library B has 6 M reads. In the heatmaps generated by DeepTools, library B's heatmap signal (read density) would be 4 times smaller compared to library A - which convolutes any biologically meaningful signal. In this repo I'll explain the bamCoverage function and recommend methods to correctly normalize your data.

Skip down to the [Summary](#summary) if you just want the TLDR.

# Relevant files and links

There are 3 main and 2 supplementary files involved in bamCoverage located in the deeptools/ directory: 

1. **bamCoverage.py**
1. **writeBedGraph.py**
1. **countReadsPerBin.py**
1. getScaleFactor.py
1. mapReduce.py

[Link](https://deeptools.readthedocs.io/en/develop/content/tools/bamCoverage.html#Output) to the original bamCoverage docs.
My in-code comments will be prefixed with '###'.

# How bamCoverage works

Once you input the following command, bamCoverage.py receives the file.

```bash
bamCoverage -b mySample.bam -o mySample.bw
```

## bamCoverage.py

```python
def get_optional_args():

    parser = argparse.ArgumentParser(add_help=False)
    optional = parser.add_argument_group('Optional arguments')

    optional.add_argument("--help", "-h", action="help",
                          help="show this help message and exit")

    optional.add_argument('--scaleFactor',
                          help='The computed scaling factor (or 1, if not applicable) will '
                          'be multiplied by this. (Default: %(default)s)',
                          default=1.0, ### if scaleFactor is not set, the libraries won't normalize by depth!
                          type=float,
                          required=False)
```

Note that if --scaleFactor is not explicitly set by the user, DeepTools will not normalize your data.

## bamCoverage.py

```python
# under def main():
    if args.normalizeUsing:
        # if a normalization is required then compute the scale factors
        bam, mapped, unmapped, stats = openBam(args.bam, returnStats=True, nThreads=args.numberOfProcessors)
        bam.close()
        scale_factor = get_scale_factor(args, stats) ### function from getScaleFactor.py
    else:
        scale_factor = args.scaleFactor
    ### calculate the scaling factor, otherwise use user's argument. by default, does not scale (scale_factor is 1.0)
``` 

If the user sets a scaling method (RPKM, CPM, BPM, RPGC, etc.) then the scaling factor will be calculated using the `getScaleFactor.py` file. That file will account for the different normalization schemes and return a scaling factor but they won't be explained in this read-through.

## bamCoverage.py

```python
# under def main():
    else:
    ### relevant code since we don't use mnase or offset.
    ### initiate writeBedGraph class object with user-provided arguments.
        wr = writeBedGraph.WriteBedGraph([args.bam],
                                         binLength=args.binSize,
                                         stepSize=args.binSize,
                                         region=args.region,
                                         blackListFileName=args.blackListFileName,
                                         numberOfProcessors=args.numberOfProcessors,
                                         extendReads=args.extendReads,
                                         minMappingQuality=args.minMappingQuality,
                                         ignoreDuplicates=args.ignoreDuplicates,
                                         center_read=args.centerReads,
                                         zerosToNans=args.skipNonCoveredRegions,
                                         samFlag_include=args.samFlagInclude,
                                         samFlag_exclude=args.samFlagExclude,
                                         minFragmentLength=args.minFragmentLength,
                                         maxFragmentLength=args.maxFragmentLength,
                                         chrsToSkip=args.ignoreForNormalization,
                                         verbose=args.verbose,
                                         )
```

If the user does not specifiy mnase or offset flags, then a writeBedGraph object will be initiated. It will take your list of bam files and user arguments, initiate a writeBedGraph object, and run the `writeBedGraph.run` method. Before we look at the run method, it's important to look at the writeBedGraph object itself, which inherits from the `countReadsPerBin` object. 

## countReadsPerBin.py

```python
class CountReadsPerBin(object):
    def __init__(self, bamFilesList, binLength=50, numberOfSamples=None, numberOfProcessors=1,
                 verbose=False, region=None,
                 bedFile=None, extendReads=False,
                 genomeChunkSize=None,
                 blackListFileName=None,
                 minMappingQuality=None,
                 ignoreDuplicates=False,
                 chrsToSkip=[],
                 stepSize=None,
                 center_read=False,
                 samFlag_include=None,
                 samFlag_exclude=None,
                 zerosToNans=False,
                 skipZeroOverZero=False,
                 smoothLength=0,
                 minFragmentLength=0,
                 maxFragmentLength=0,
                 out_file_for_raw_data=None,
                 bed_and_bin=False,
                 statsList=[],
                 mappedList=[]):

        self.bamFilesList = bamFilesList
        self.binLength = binLength
        self.numberOfSamples = numberOfSamples
        self.blackListFileName = blackListFileName
        self.statsList = statsList
        self.mappedList = mappedList
        self.skipZeroOverZero = skipZeroOverZero
        self.bed_and_bin = bed_and_bin
        self.genomeChunkSize = genomeChunkSize

        if extendReads and len(bamFilesList):
            from deeptools.getFragmentAndReadSize import get_read_and_fragment_length
            frag_len_dict, read_len_dict = get_read_and_fragment_length(bamFilesList[0],
                                                                        return_lengths=False,
                                                                        blackListFileName=blackListFileName,
                                                                        numberOfProcessors=numberOfProcessors,
                                                                        verbose=verbose)
            if extendReads is True:
                # try to guess fragment length if the bam file contains paired end reads
                if frag_len_dict:
                    self.defaultFragmentLength = int(frag_len_dict['median'])
                else:
                    exit("*ERROR*: library is not paired-end. Please provide an extension length.")
                if verbose:
                    print(("Fragment length based on paired en data "
                          "estimated to be {}".format(frag_len_dict['median'])))

            elif extendReads < read_len_dict['median']:
                sys.stderr.write("*WARNING*: read extension is smaller than read length (read length = {}). "
                                 "Reads will not be extended.\n".format(int(read_len_dict['median'])))
                self.defaultFragmentLength = 'read length'

            elif extendReads > 2000:
                exit("*ERROR*: read extension must be smaller that 2000. Value give: {} ".format(extendReads))
            else:
                self.defaultFragmentLength = int(extendReads)

        else:
            self.defaultFragmentLength = 'read length'

        self.numberOfProcessors = numberOfProcessors
        self.verbose = verbose
        self.region = region
        self.bedFile = bedFile
        self.minMappingQuality = minMappingQuality
        self.ignoreDuplicates = ignoreDuplicates
        self.chrsToSkip = chrsToSkip
        self.stepSize = stepSize
        self.center_read = center_read
        self.samFlag_include = samFlag_include
        self.samFlag_exclude = samFlag_exclude
        self.minFragmentLength = minFragmentLength
        self.maxFragmentLength = maxFragmentLength
        self.zerosToNans = zerosToNans
        self.smoothLength = smoothLength

        if out_file_for_raw_data:
            self.save_data = True
            self.out_file_for_raw_data = out_file_for_raw_data
        else:
            self.save_data = False
            self.out_file_for_raw_data = None

        # check that wither numberOfSamples or stepSize are set
        if numberOfSamples is None and stepSize is None and bedFile is None:
            raise ValueError("either stepSize, numberOfSamples or bedFile have to be set")

        if self.defaultFragmentLength != 'read length':
            self.maxPairedFragmentLength = 4 * self.defaultFragmentLength
        else:
            self.maxPairedFragmentLength = 1000
        if self.maxFragmentLength > 0:
            self.maxPairedFragmentLength = self.maxFragmentLength

        if len(self.mappedList) == 0:
            try:
                for fname in self.bamFilesList:
                    bam, mapped, unmapped, stats = bamHandler.openBam(fname, returnStats=True, nThreads=self.numberOfProcessors)
                    self.mappedList.append(mapped)
                    self.statsList.append(stats)
                    bam.close()
            except:
                self.mappedList = []
                self.statsList = []

```

The above code will store the user's arguments into the object's attributes, extend reads, determine output file name, and make sure bin size and window size make sense. Storing user arguments to the object is similar to storing books into the slots on a bookshelf.

## countReadsPerBin.py

```python
    def run(self, allArgs=None):
        ### Open all bam files list.
        bamFilesHandles = []
        for x in self.bamFilesList:
            try:
                y = bamHandler.openBam(x)
            except SystemExit:
                sys.exit(sys.exc_info()[1])
            except:
                y = pyBigWig.open(x)
            bamFilesHandles.append(y)

        chromsizes, non_common = deeptools.utilities.getCommonChrNames(bamFilesHandles, verbose=self.verbose)

        # skip chromosome in the list. This is usually for the
        # X chromosome which may have either one copy  in a male sample
        # or a mixture of male/female and is unreliable.
        # Also the skip may contain heterochromatic regions and
        # mitochondrial DNA
        if len(self.chrsToSkip):
            chromsizes = [x for x in chromsizes if x[0] not in self.chrsToSkip]

        chrNames, chrLengths = list(zip(*chromsizes))

        genomeSize = sum(chrLengths)

        chunkSize = None
        if self.bedFile is None:
            if self.genomeChunkSize is None:
                chunkSize = self.get_chunk_length(bamFilesHandles, genomeSize, chromsizes, chrLengths) ### a chunk is 
            else:
                chunkSize = self.genomeChunkSize

        [bam_h.close() for bam_h in bamFilesHandles]

        if self.verbose:
            print("step size is {}".format(self.stepSize))

        if self.region:
            # in case a region is used, append the tilesize
            self.region += ":{}".format(self.binLength)

        # Handle GTF options
        transcriptID, exonID, transcript_id_designator, keepExons = deeptools.utilities.gtfOptions(allArgs)

        # use map reduce to call countReadsInRegions_wrapper
        ### purpose of mapReduce is to apply the countReadsInRegions_wrapper function with multiprocessing
        ### countReadsInRegions_wrapper used to invoke the cr.count_reads_in_region function in accordance to multiprocessing module.
        ### this will generate intermediate bedgraph files, which will be concatenated together.
        imap_res = mapReduce.mapReduce([],
                                       countReadsInRegions_wrapper,
                                       chromsizes,
                                       self_=self,
                                       genomeChunkLength=chunkSize,
                                       bedFile=self.bedFile,
                                       blackListFileName=self.blackListFileName,
                                       region=self.region,
                                       numberOfProcessors=self.numberOfProcessors,
                                       transcriptID=transcriptID,
                                       exonID=exonID,
                                       keepExons=keepExons,
                                       transcript_id_designator=transcript_id_designator)

        if self.out_file_for_raw_data:
            if len(non_common):
                sys.stderr.write("*Warning*\nThe resulting bed file does not contain information for "
                                 "the chromosomes that were not common between the bigwig files\n")

            # concatenate intermediary bedgraph files
            ofile = open(self.out_file_for_raw_data, "w")
            for _values, tempFileName in imap_res:
                if tempFileName:
                    # concatenate all intermediate tempfiles into one
                    _foo = open(tempFileName, 'r')
                    shutil.copyfileobj(_foo, ofile)
                    _foo.close()
                    os.remove(tempFileName)

            ofile.close()

        try:
            num_reads_per_bin = np.concatenate([x[0] for x in imap_res], axis=0)
            return num_reads_per_bin

        except ValueError:
            if self.bedFile:
                sys.exit('\nNo coverage values could be computed.\n\n'
                         'Please check that the chromosome names in the BED file are found on the bam files.\n\n'
                         'The valid chromosome names are:\n{}'.format(chrNames))
            else:
                sys.exit('\nNo coverage values could be computed.\n\nCheck that all bam files are valid and '
                         'contain mapped reads.')
```

The CountReadsPerBin object has its own `run` method as well. It first creates file handles to all bam files, removes any chromosomes to skip, determines a genome chunk length, and reads a GTF file if provided before closing the file handles. The chunk size determines how much genome to send to workers to efficiently use multiple threads. A genome chunk length is defined as an "optimal fraction of the genome (chunkSize) that is sent to workers for analysis. If too short, too much time is spent loading the files if too long, some processors end up free". By default with no user specification, the chunk size is (stepSize * 1000) / (max mapped reads / genome size * number of samples).

Once all the important parameters are calculated, DeepTools will count reads in region with the `count_reads_in_region` function. The `count_reads_in_region` function is the mechanism for read counting, and the binSize and stepSize parameters here will define the windows. The relationship between binSize and stepSize are perfectly summarized in the code itself:

>        The stepSize controls the distance between bins. For example,
>        a step size of 20 and a bin size of 20 will create bins next to
>        each other. If the step size is smaller than the bin size the
>        bins will overlap.

To utilize the the count_reads_in_region function, it is passed to the `mapReduce` function which uses the count_reads_in_region_wrapper function; something like this: mapReduce --> count_reads_in_region_wrapper --> count_reads_in_region. mapReduce will interface with the multiprocessing python module to parallelize read counting - whether it parallelizes across samples or one sample at a time, I did not read that closely. **The output of count_reads_in_region is a numpy array where rows are genomic bins and columns are samples.** Multiple intermediate bedgraphs (the output of count_reads_in_region) will be concatenated together. In the context of data normalization, this detail is directly relevant but still interesting nonetheless.

Now that we know the CountReadsPerBin class stores important user parameters and performs read counting, we will investigate what is unique about the writeBedGraph class.

## bamCoverage.py

But wait there's more! The following are the arguments passed to writeBedGraph.run method, and will be important in how DeepTools scales coverage. 

```python
# under def main(), last bits of code

    ### utilize the run method in writeBedGraph class.
    ### func_args is the scale factor dictionary.
    ### writeBedGraph.scaleCoverage is default function to call
    ### bamCoverage.py --> writeBedGraph.py
    wr.run(writeBedGraph.scaleCoverage, func_args, args.outFileName,
           blackListFileName=args.blackListFileName,
           format=args.outFileFormat, smoothLength=args.smoothLength)
```

## writeBedGraph.py

In the writeBedGraph, there are no special attributes! Just new methods on how to scale coverage and output bigwig files.

```python
    def run(self, func_to_call, func_args, out_file_name, blackListFileName=None, format="bedgraph", smoothLength=0):
        r"""
        Given a list of bamfiles, a function and a function arguments,
        this method writes a bedgraph file (or bigwig) file
        for a partition of the genome into tiles of given size
        and a value for each tile that corresponds to the given function
        and that is related to the coverage underlying the tile.

        Parameters
        ----------
        func_to_call : str
            function name to be called to convert the list of coverages computed
            for each bam file at each position into a single value. An example
            is a function that takes the ratio between the coverage of two
            bam files.
        func_args : dict
            dict of arguments to pass to `func`. E.g. {'scaleFactor':1.0}

        out_file_name : str
            name of the file to save the resulting data.

        smoothLength : int
            Distance in bp for smoothing the coverage per tile.


        """
        self.__dict__["smoothLength"] = smoothLength
        getStats = len(self.mappedList) < len(self.bamFilesList)
        ### Load all bam handles
        bam_handles = []
        for x in self.bamFilesList:
            if getStats:
                bam, mapped, unmapped, stats = bamHandler.openBam(x, returnStats=True, nThreads=self.numberOfProcessors)
                self.mappedList.append(mapped)
                self.statsList.append(stats)
            else:
                bam = bamHandler.openBam(x)
            bam_handles.append(bam)

        genome_chunk_length = getGenomeChunkLength(bam_handles, self.binLength, self.mappedList)
        # check if both bam files correspond to the same species
        # by comparing the chromosome names:
        chrom_names_and_size, non_common = getCommonChrNames(bam_handles, verbose=False)

        if self.region:
            # in case a region is used, append the tilesize
            self.region += ":{}".format(self.binLength)

        for x in list(self.__dict__.keys()):
            if x in ["mappedList", "statsList"]:
                continue
            sys.stderr.write("{}: {}\n".format(x, self.__getattribute__(x)))

        ### call writeBedGraph_worker which will calculate + smooth coverage across multiple threads. 
        res = mapReduce.mapReduce([func_to_call, func_args],
                                  writeBedGraph_wrapper,
                                  chrom_names_and_size,
                                  self_=self,
                                  genomeChunkLength=genome_chunk_length,
                                  region=self.region,
                                  blackListFileName=blackListFileName,
                                  numberOfProcessors=self.numberOfProcessors)

        # Determine the sorted order of the temp files
        chrom_order = dict()
        for i, _ in enumerate(chrom_names_and_size):
            chrom_order[_[0]] = i
        res = [[chrom_order[x[0]], x[1], x[2], x[3]] for x in res]
        res.sort()
        
        if format == 'bedgraph':
            out_file = open(out_file_name, 'wb')
            for r in res:
                if r[3]:
                    _foo = open(r[3], 'rb')
                    shutil.copyfileobj(_foo, out_file)
                    _foo.close()
                    os.remove(r[3])
            out_file.close()
        else:
            bedGraphToBigWig(chrom_names_and_size, [x[3] for x in res], out_file_name)
            ### aggreagate bedgraph files into bigwig file.
``` 

Similar to the CountReadsPerBin run method, the writeBedGraph.run method will create file handles to all bam files, calculate chunk length, and use mapReduce to call writeBedGraph_wrapper to utilize writeBedGraph_worker which counts reads + smooths signal via CountReadsPerBin.count_reads_in_region. Chromosome order is then determined, and the bedGraphToBigWig function is called to take a sorted list of bedgraph files and write into one bigwig file using pyBigWig by adding 1 M entries at a time.

In summary, writeBedGraph.run is summarized as: mapReduce --> writeBedGraph_wrapper --> writeBedGraph_worker --> CountReadsPerBin.count_reads_in_region

At this stage, read normalization is performed in the writeBedGraph.writeBedGraph_worker method: 

## writeBedGraph_worker

```python
# from writeBedGraph_worker

        if start > end:
            raise NameError("start position ({0}) bigger "
                            "than end position ({1})".format(start, end))

        coverage, _ = self.count_reads_in_region(chrom, start, end) ### calculate the coverage in each bam file at each stepSize

        _file = open(utilities.getTempFileName(suffix='.bg'), 'w') ### open a bedgraph file to write to.
        previous_value = None
        line_string = "{}\t{}\t{}\t{:g}\n"
        for tileIndex in range(coverage.shape[0]):

            if self.smoothLength is not None and self.smoothLength > 0:
                vector_start, vector_end = self.getSmoothRange(tileIndex,
                                                               self.binLength,
                                                               self.smoothLength,
                                                               coverage.shape[0])
                tileCoverage = np.mean(coverage[vector_start:vector_end, :], axis=0)
            else:
                tileCoverage = coverage[tileIndex, :]
            if self.skipZeroOverZero and np.sum(tileCoverage) == 0:
                continue

            value = func_to_call(tileCoverage, func_args)

```

The numpy array of coverage generated by count_reads_in_region is stored in the `coverage` variable. If no smoothing is called, then the `tileCoverage` variable is the same array. The `value` variable stores normalized reads, and the func_to_call is replaced by the writeBedGraph.scaleCoverage function defined as: 

```python
def scaleCoverage(tile_coverage, args):
    """
    tileCoverage should be an list with only one element
    """
    return args['scaleFactor'] * tile_coverage[0]
``` 

Which simply multiplies the user argument's scaling factor by a bin's coverage. So if a bin has 100 reads and the user specifies a scale of 0.25, then 0.25 * 100 = 25 reads. But without specifying a scaling factor, the default behavior is to set scaleFactor to 1.0 which does not normalize your data. In conclusion, normalization in this context is basically multiplying a scalar (scaleFactor) by your signal (reads per bin).

# Summary

DeepTools bamCoverage will take your bam file(s) and count the reads in bins (e.g. 10 bp) while leveraging multiple threads if provided. Once the coverage per bin is determined, the user-provided scaling factor is multiplied by the coverage. When all the reads have been counted across genome chunks, all intermediate bedgraph files are aggregated + formatted into one bigwig file. Not normalizing bigwig files can confound any interesting biological signals of interest. For example, library A can have twice the signal as library B simply because A is sequenced deeper.

**You should normalize your sequencing libraries if they have non-uniform depth. Not normalizing by depth will affect your figures!**

# How to correctly normalize your data 1

There are two methods to normalize your data, but first they require a couple of pre-requisite steps: 

## Count number of reads in your bam file

```bash
touch reads.txt

for i in $(find /path/to/bam/files -name "*.bam" | sort); do 
    sample=$(basename $i | cut -d "." -f1)
    reads=$(samtools view -@ 4 -c $i)
    echo -e "$sample\t$reads"
    echo -e "$sample\t$reads" >> reads.txt
done
# find will find your bam files. -name flag finds all files ending in ".bam" and list of files is sorted
# save sample name and number of reads into variables. basename function will strip all upstream file names, and cut retains text before first "."
# echo sample and number of reads into stdout and reads.txt
```

The reads.txt file should look like this: 

```
DoxB_H3K27Ac    34020742
DoxB_H3K4me1    43255614
DoxB_H3K4me3    45376160
DoxWdB_H3K27Ac  17834062
DoxWdB_H3K4me1  37854552
DoxWdB_H3K4me3  45515910
```

## Calculate scaling ratios

Then select the sample with the smallest library and divide the entire column by that number. Calculating the numbers in excel also works too.

```bash
min_read=$(cut -f2 reads.txt | sort -n | head -1) # smallest library, a constant

awk -v min_read="$min_read" -v OFS="\t" '{ print $0, min_read/$2*100 }' reads.txt > reads.2.txt

DoxB_H3K27Ac    34020742        52.4211
DoxB_H3K4me1    43255614        41.2295
DoxB_H3K4me3    45376160        39.3027
DoxWdB_H3K27Ac  17834062        100
DoxWdB_H3K4me1  37854552        47.1121
DoxWdB_H3K4me3  45515910        39.182
```

With the `reads.2.txt` file, you can move onto the next step.

# How to correctly normalize your data 2

## Random down-sample via samtools

Downsample all libraries using samtools view -s flag prior to using bamCoverage.

```bash
for i in $(find . -name "*.bam" | sort); do
    sample=$(basename $i | cut -d "." -f1)
    ratio=$(grep $sample reads.2.txt | cut -f3)


    if [[ "$ratio" == 100 ]]; then
        echo $sample is lowest-sequenced library.
        echo "cp $i $sample.downsample.bam"
        cp $i $sample.downsample.bam
        continue
    fi


    echo -e "samtools view -bS -@ 2 -s 0.$ratio $i > $sample.downsample.bam"
    samtools view -bS -@ 4 -s 0.$ratio $i > $sample.downsample.bam &
done
# ratio is a sample's ratio in reads.2.txt
# if the ratio is 100 simply copy + rename the smallest library.
# -s 0.$ratio means seed of 0 and subsample $ratio (e.g. 42%) number of reads from the input bam file.
# The & puts a command to the background to parallelize work. 
```

The output is a list of bam files whose depth matches the lowest-sequenced library.

```bash
for i in $(find . -name "*downsample*"| sort); do 
    sample=$(basename $i)
    counts=$(samtools view -@ 4 -c $i)
    echo -e "$sample\t$counts"
done

DoxB_H3K27Ac.downsample.bam     17684452
DoxB_H3K4me1.downsample.bam     17739786
DoxB_H3K4me3.downsample.bam     17698772
DoxWdB_H3K27Ac.downsample.bam   17834062
DoxWdB_H3K4me1.downsample.bam   17789764
DoxWdB_H3K4me3.downsample.bam   17748524
```

With the down-sampled bam files, make sure to index them all and continue through bamCoverage to generate your bigwig files. **When using bamCoverage this time, you do not need to set the `--scaleFactor` flag since you've already normalized everything!**

An example bamCoverage call would be like the following. Notice it doesn't have the --scaleFactor argument.

```bash
bamCoverage -b <input> -o <output> -p 8 --binSize 10
```

## Use the --scaleFactor flag in DeepTools

Define the scaling factor by adding the `--scaleFactor`. Only the reads.2.txt file is needed for this method.

```bash
# load up a conda environment with deeptools installed
mkdir bw

for i in $(find /path/to/bam/file -name "*.bam"); do
    sample=$(basename $i | cut -d "." -f1)
    ratio=$(grep $sample reads.2.txt | cut -f3)
    scaleFactor=$(bc -l <<< $ratio/100)
    echo "bamCoverage -b $i -o bw/$sample.bw -p 16 --binSize 10 --scaleFactor $scaleFactor -v"
    srun -N 1 --cpus-per-task 16 -p exacloud --time=01:00:00 bamCoverage -b $i -o bw/$sample.bw -p 16 --binSize 10 --scaleFactor $scaleFactor -v &
done
# scaleFactor is just ratio but divided by 100
# srun pushes the command to exacloud servers with the listed resources.
# -v for verbose. Please look at scaleFactor value on the screen to double check!
```

DeepTools will multiply the scaleFactor by the coverage, which is unique to each library. The output in this case will be a list of bigwig files. You can check the status of your jobs by typing `watch squeue -u $(whoami)` and use "Ctrl+C" to quit.



The output of either method are sequencing-depth normalized bigwig files, which will be appropriate for downstream data visualization with IGV and DeepTools computeMatrix + plotHeatmap.