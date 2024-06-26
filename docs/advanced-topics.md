# Advanced topics

## Reference genome

In some cases, DoubleHelix needs to access the reference genome that was used to create an alignment-map file. Since this information is not contained in an alignment-map file, DoubleHelix needs to guess it. This document explains how DoubleHelix solves this problem and what can go wrong.

### Introduction
The biggest hint for identifying the reference genome is the header of an alignment-map file. The header contains a list of sequences contained in the reference used to do the alignment.

DoubleHelix solves the issue of identifying the reference by maintaining a list of meta-information about several reference genomes. In this way, it can identify a reference if it's present in this list. The list can be modified and new references can be suggested by using this [GitHub issue template](https://github.com/DoubleHelixApp/DoubleHelix/issues/new?assignees=chaplin89&labels=reference&projects=&template=add-a-new-reference.md&title=%5BReference%5D+Please+add+a+new+reference) or by submitting a PR using the instruction below.

The process is not perfect and it can potentially give incorrect information (see the section below to understand how it works). In this case, some actions may fail. Please feel free to open a [bug report](https://github.com/DoubleHelixApp/DoubleHelix/issues/new?assignees=&labels=&projects=&template=bug_report.md&title=) if that happens.

### How it works

The identification can work in two different ways, depending on the information available in the alignment-map file: 

- Using MD5 of sequences 
- Using lengths of sequences

MD5 indicates an MD5 string calculated in a reliable way (that accounts for case differences inside the sequence) over the content of a sequence. The procedure to calculate the MD5 is described on page 6 of the SAM standard. An MD5 can identify unequivocally a specific sequence and it's the preferred method to find a reference. If MD5 of the reference sequences are available it's easy to find the reference looking in the meta-data list of reference genomes that DoubleHelix maintains. Lengths are used only as a fallback when MD5s are not available. Unfortunately, this situation is the most common one. Despite MD5 sequences being part of the SAM specification standard they are not mandatory. Most alignment-map files won't have this field populated.

The lengths indicate the length of the sequences expressed in base pairs. A length itself cannot reliably identify a sequence as it's possible (and happening in practices) to have the same lengths for sequences having completely different content (and hence a different MD5). What's more reliable is to use the whole set of sequences to identify a reference. The sequence of lengths contained in a reference is pretty unique for each reference, and it's the current way used by DoubleHelix to identify a reference when the MD5 is not available. It's still possible that two references have a perfectly identical set of lengths having a different sequence of MD5s. In this case, if the alignment-map file was associated by DoubleHelix to a reference falling in this situation, DoubleHelix will present a choice to the user. Since it's impossible to reliably determine which reference was used without other information, the user has to choose which reference is the correct one.

This is a list of ambiguous lengths sequences (will be updated at  every commit)

1. https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/analysisSet/hg38.analysisSet.fa.gz
1. https://ftp.ncbi.nlm.nih.gov/genomes/archive/old_genbank/Eukaryotes/vertebrates_mammals/Homo_sapiens/GRCh38/seqs_for_alignment_pipelines/GCA_000001405.15_GRCh38_no_alt_analysis_set.fna.gz


1. https://ftp.ensembl.org/pub/release-55/fasta/homo_sapiens/dna/Homo_sapiens.GRCh37.55.dna.toplevel.fa.gz
2. https://ftp.ensembl.org/pub/release-56/fasta/homo_sapiens/dna/Homo_sapiens.GRCh37.56.dna.toplevel.fa.gz

### Add a new reference

It's possible to onboard a new reference to DoubleHelix either by opening a GitHub issue with [this template](https://github.com/DoubleHelixApp/DoubleHelix/issues/new?assignees=chaplin89&labels=reference&projects=&template=add-a-new-reference.md&title=%5BReference%5D+Please+add+a+new+reference) or by using the following code:

```python
manager = RepositoryManager()
manager.genomes.append(
    manager.ingest(
        "https://source/reference.fa",
        "NIH", # Anything that matches an entry in sources.json, otherwise add an entry there
        "38",  # Only 38 or 19
    )
)
GenomeMetadataLoader().save(manager.genomes)
```

A GUI/CLI way will be added in the future.


## Progress bar

To provide a friendlier interface, DoubleHelix shows a progress bar while doing long operations. Implementing a progress bar for DoubleHelix required a considerable effort, as DoubleHelix is reliant on dependencies who doesn't allow to monitor their progress in any way.

In these cases, the approach DoubleHelix is following is to estimate how many bytes will the process needs to read or write and constantly polling the current amount of bytes processed to understand where the process is. Is not easy to estimate this info in some cases, and in general the process is quite hacky, meaning it can break easily (giving incorrect ETA or percentage) after advancing the version of these dependencies. Some classes to abstract this behaviour are making this process as painless as possible.

The architecture like this: ProcessIOMonitor -> BaseProgressCalculator -> UI Callback

All these classes are in the `helix.progress` namespace:

- *helix.progress.ProcessIOMonitor*: will start a process and preiodically calling a callback, supplying [`io_counters`](https://psutil.readthedocs.io/en/latest/#psutil.disk_io_counters) (i.e., read/write bytes from the process).
- *helix.progress.BaseProcessCalculator*: Implements the callback for the class above. `BaseProgressCalculator` knows what's the target to reach and convert the bytes supplied by `ProcessIOMonitor` into a percentage and an ETA. It implements all the features needed to have stable values.
- *helix.progress.FileIOMonitor*: Is the equivalent of ProcessIOMonitor but it monitors read or write bytes on a file, not on the process itself. Sometime is simply more practical to monitor the growth of a file, as a process may not exists (i.e., the file is generate by another 3rd party python module that does not provide any way to monitor the progress). It provides an interface that is similar to `ProcessIOMonitor`.