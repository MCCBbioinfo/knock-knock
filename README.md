# knock-knock

`knock-knock` is a tool for exploring, categorizing, and quantifying the sequence outcomes produced by CRISPR knock-in experiments.

![](docs/table_demo.gif)

knock-knock organizes all references sequences, sequencing data, and analysis output for a given project inside of a project directory (e.g. `/PROJECT_DIR`).
Every time knock-knock is run, this directory is given as the first command line argument to tell knock-knock which project to analyze.
Inside a project directory, a directory called `targets` holds information about each locus that was targeted for editing.
To build these targets, create a csv '/PROJECT_DIR/targets/targets.csv` with a header row of `name,sgRNA_sequence,amplicon_primers,donor_sequence,donor_primers,nonhomologous_donor_sequence` and a row providing the corresponding information for each target locus. 
 Details for each field:
- `sgRNA_sequence` is the protospacer sequence that was cut (IMPORTANT: 5' to 3' and not including the PAM). If there are multiple guides, include all sgRNAs sequence joined by semi colons.
- `amplicon_primers` are the primers used to generate the amplicon for sequencing, with both given as the actual 5' to 3' sequence of the primers themselves (i.e. so the two are on opposite strands) but only including genomic sequence (not e.g. any sequencing adapters), given as two strings with a semicolon in between them.
- `donor_sequence` is 
- `donor_primers` are 
- `nonhomologous_donor_sequence` is

After supplying this csv, run ```knock-knock /PROJECT_DIR build_targets``` to convert the details you have added into the form needed for subsequent analysis.

Fastqs should be put in a directory called `data` inside the base_dir, with each group of experiments (typically from one sequencing run) stored in a further subdirectory (e.g. `/PROJECT_DIR/data/EXAMPLE_RUN`).
Each group directory needs a sample sheet to tell knock-knock which target to align each experiment to.
To provide this, make a csv called `/PROJECT_DIR/data/EXAMPLE_RUN/sample_sheet.csv` in the group subdirectory with header row `sample,platform,R1,R2,CCS_fastq_fns,target_info,donor`

where:
- sample is the sample name
- platform is 'illumina' or 'pacbio'
- if platform is 'illumina', R1 and R2 are a pair of R1 and R2 fastq file names in the group's directory (not the full path, just the part after the last slash)
- if platform is 'pacbio', CCS_fastq_fns are circular consensus sequence fastq file names (not the full path, just the part after the last slash)
- target_info is the name of an editing target that exists in this project's target directory
- donor is 
- color (optional) is a number you can provide such that all samples with the same number will be given the same color in the html tables


Example directory structure:
```
base_dir
├── data
│   ├── group1
│   │   ├── sample_sheet.yaml
│   │   ├── data1.fastq
│   │   └── data2.fastq
│   └── group1
│       ├── sample_sheet.yaml
│       ├── data1.fastq
│       └── data2.fastq
└── targets
    └── locus_name
        ├── manifest.yaml
        ├── refs.fasta
        ├── refs.fasta.fai
        └── refs.gff
```
Input fastqs and sample sheets describing the experiments that produced them are kept in base_dir/data in 'group' directories.

Each sample_sheet.yaml should contain {experiment names: {description dictionary}} pairs.
Each experiment's description dictionary must always contain the key-value pair:

    target_info: name of the target (see below)

For a PacBio experiment, it should also contain

    fastq_fn: name of the fastq file (relative to the group's directory, i.e. just the basename)

For a MiSeq experiment, it should contain:
    R1_fn: name of the R1 fastq file (relative to the group's directory, i.e. just the basename)
    R2_fn: name of the R2 fastq file (relative to the group's directory, i.e. just the basename)
    
It can also contain other optional keys describing properties of the experiment, e.g.:
    cell_line: string
    donor_type: string
    replicate: integer
    sorted: boolean

target_info should be the name of a directory in base_dir/targets (e.g. locus_name above) containing descriptions of the sequences used in the experiment.
Nomenclature: 'target' is the genomic locus being targeted, and 'donor' is the exogenous sequence being added.

The required files are:

- a fasta file 'refs.fasta' containing the target sequence, donor sequence, and any suspected contaminants

- an index 'refs.fasta.fai' of this fasta file

- a gff file 'refs.gff' marking various features on these sequences.

    Required features are -
    target: [sgRNA, 5' HA, 3' HA, forward primer, reverse primer]
    donor: [5' HA, 3' HA] (TODO: this is incomplete)

- 'manifest.yaml', with contents

    target: seqname in refs.fasta of target
    donor: seqname in refs.fasta of donor
    sources: list of genbank files from which to generate refs.\*

The fasta, fai, and gff  files can be generated by annotating the target and donor sequences with the required features in benchling, downloading the benchling files in genbank format into the locus_name directory, listing these files in the 'sources' entry in 'manifest.yaml', then running TargetInfo.make_references (TODO: explain).

Once an experiment_name directory containing data and a manifest pointing to a valid target_info directory exists, process the experiment with 

    knock-knock --base_dir {base_dir} process group_name experiment_name

If GNU parallel is installed, you can run

    knock-knock --base_dir {base_dir} parallel {max_procs}

to process all experiments that exist in base_dir in parallel up to max_procs at a time.

To select only a subset of experiments to process, 

    knock-knock --base_dir {base_dir} --parallel {max_procs} --condition "yaml here"

where the yaml gives conditions that the optional properties of an experiment need to satisify. (TODO: explain)

After experiments have been processed, to generate csv and html tables of outcome counts:
    
    knock-knock --base_dir {base_dir} table {file_name_prefix}

The 'table' command can be given the same kind of --condition argument to restrict to a subset.

To explore outcomes in more depth than the html table, run

    import knock_knock.visualize
    knock_knock.visualize.explore(your_base_dir, by_outcome=True)

in a Jupyter notebook.
