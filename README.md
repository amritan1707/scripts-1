# Scripts for training and validating neural network models for gnina

 * train.py - Takes a model template and train/test file(s)
 * predict.py - Takes a model, learned weights, and an input file and outputs probabilities of binding
 * clustering.py - Takes an input file, and outputs clustered cross-validation train/test file(s) for train.py. Note: take a long time to compute
 * compute_seqs.py - Takes input file for clustering.py and creates the input for compute_row.py
 * compute_row.py - Computes 1 row of NxN matrix in clustering.py. This allows for more parallelization of clustering.py
 * combine_rows.py - Script to take outputs of compute_row.py & combine them to avoid needing to do the computation in clustering.py

## Dependencies

```
sudo pip install matplotlib scipy sklearn scikit-image protobuf psutil numpy seaborn
export PYTHONPATH=/usr/local/python:$PYTHONPATH
```
rdkit -- see installation instructions [here](https://www.rdkit.org/docs/Install.html)

## Training
```
usage: train.py [-h] -m MODEL -p PREFIX [-d DATA_ROOT] [-n FOLDNUMS] [-a]
                [-i ITERATIONS] [-s SEED] [-t TEST_INTERVAL] [-o OUTPREFIX]
                [-g GPU] [-c CONT] [-k] [-r] [--avg_rotations] [--keep_best]
                [--dynamic] [--cyclic] [--solver SOLVER] [--lr_policy LR_POLICY]
                [--step_reduce STEP_REDUCE] [--step_end STEP_END]
                [--step_when STEP_WHEN] [--base_lr BASE_LR]
                [--momentum MOMENTUM] [--weight_decay WEIGHT_DECAY]
                [--gamma GAMMA] [--power POWER] [--weights WEIGHTS]
                [-p2 PREFIX2] [-d2 DATA_ROOT2] [--data_ratio DATA_RATIO]

Train neural net on .types data.

optional arguments:
  -h, --help            show this help message and exit
  -m MODEL, --model MODEL
                        Model template. Must use TRAINFILE and TESTFILE
  -p PREFIX, --prefix PREFIX
                        Prefix for training/test files:
                        <prefix>[train|test][num].types
  -d DATA_ROOT, --data_root DATA_ROOT
                        Root folder for relative paths in train/test files
  -n FOLDNUMS, --foldnums FOLDNUMS
                        Fold numbers to run, default is '0,1,2'
  -a, --allfolds        Train and test file with all data folds,
                        <prefix>.types
  -i ITERATIONS, --iterations ITERATIONS
                        Number of iterations to run,default 10,000
  -s SEED, --seed SEED  Random seed, default 42
  -t TEST_INTERVAL, --test_interval TEST_INTERVAL
                        How frequently to test (iterations), default 40
  -o OUTPREFIX, --outprefix OUTPREFIX
                        Prefix for output files, default <model>.<pid>
  -g GPU, --gpu GPU     Specify GPU to run on
  -c CONT, --cont CONT  Continue a previous simulation from the provided
                        iteration (snapshot must exist)
  -k, --keep            Don't delete prototxt files
  -r, --reduced         Use a reduced file for model evaluation if exists(<pre
                        fix>[_reducedtrain|_reducedtest][num].types)
  --avg_rotations       Use the average of the testfile's 24 rotations in its
                        evaluation results
  --keep_best           Store snapshots everytime test AUC improves
  --dynamic             Attempt to adjust the base_lr in response to training
                        progress
  --cyclic		Vary base_lr between fixed values based on test 
			iteration
  --solver SOLVER       Solver type. Default is SGD
  --lr_policy LR_POLICY
                        Learning policy to use. Default is inv.
  --step_reduce STEP_REDUCE
                        Reduce the learning rate by this factor with dynamic
                        stepping, default 0.5
  --step_end STEP_END   Terminate training if learning rate gets below this
                        amount
  --step_when STEP_WHEN
                        Perform a dynamic step (reduce base_lr) when training
                        has not improved after this many test iterations,
                        default 10
  --base_lr BASE_LR     Initial learning rate, default 0.01
  --momentum MOMENTUM   Momentum parameters, default 0.9
  --weight_decay WEIGHT_DECAY
                        Weight decay, default 0.001
  --gamma GAMMA         Gamma, default 0.001
  --power POWER         Power, default 1
  --weights WEIGHTS     Set of weights to initialize the model with
  -p2 PREFIX2, --prefix2 PREFIX2
                        Second prefix for training/test files for combined
                        training: <prefix>[train|test][num].types
  -d2 DATA_ROOT2, --data_root2 DATA_ROOT2
                        Root folder for relative paths in second train/test
                        files for combined training
  --data_ratio DATA_RATIO
                        Ratio to combine training data from 2 sources
  --test_only           Don't train, just evaluate test nets once
```

MODEL is a caffe model file and is required. It should have a MolGridDataLayer
for each phase, TRAIN and TEST. The source parameter of these layers should
be placeholder values "TRAINFILE" and "TESTFILE" respectively.

PREFIX is the prefix of pre-specified training and test files.  For example, 
if the prefix is "all" then there should be files "alltrainX.types" and
"alltestX.types" for each X in FOLDNUMS.
FOLDNUMS is a comma-separated list of ints, for example 0,1,2.
With the --allfolds flag set, a model is also trained and tested on a single file
that hasn't been split into train/test folds, for example "all.types" in the
previous example.

The model trained on "alltrain0.types" will be tested on "alltest0.types".
Each model is trained for up to ITERATIONS iterations and tested each TEST_INTERVAL
iterations.

The train/test files are of the form
    1 set1/159/rec.gninatypes set1/159/docked_0.gninatypes
where the first value is the label, the second the receptor, and the third
the ligand.  Additional whitespace delimited fields are ignored.  gninatypes
files are created using gninatyper.
The receptor and label paths in these files can be absolute, or they can be
relative to a path provided as the DATA_ROOT argument. To use this option,
 the root_folder parameter in each MolGridDataLayer of the model file should be
the placeholder value "DATA_ROOT". This can also be hard-coded into the model.



The prefix of all generated output files will be OUTPREFIX.  If not specified,
it will be the model name and the process id.  While running, OUTPREFIX.X.out files are generated.
A line is written at every TEST_INTERVAL and consists of the test auc, the train auc, the loss,
and the learning rate at that iteration.

The entire train and test sets are evaluated every TEST_INTERVAL.  If they are very large
this may be undesirable.  Alternatively, if -r is passed, pre-specified reduced train/test sets
can be used for monitoring.

Once each fold is complete OUTPREFIX.X.finaltest is written with the final predicted values.

After the completion of all folds, OUTPREFIX.test and OUTPREFIX.train are written which
contain the average AUC and and individual AUCs for each fold at eash test iteration.
Also, a total finaltest file of all the predictions.  Graphs of the training behavior
(OUTPREFIX_train.pdf) and final ROC (OUTPREFIX_roc.pdf) are also created as well as caffe files.

The GPU to use can be specified with -g.

Previous training runs can be continued with -c.  The same prefix etc. should be used.

A large number of training hyperparameters are available as options.  The defaults should be pretty reasonable.

## Generating Clustered-Cross Validation splits of data

There are 2 strategies: 1) Running clustering.py directly; 2) Running compute_seqs.py --> compute_row.py --> combine_rows.py.
Strategy 1 works best when there is a small number of training examples (around 4000), but the process is rather slow.
Strategy 2 is to upscale computing the clusters (typically for a supercomputing cluster) where each row would correspond to 1 job.

Note that these scripts assume that the input files point to a relative path from the current working directory.

### Case 1: Using clustering.py
```
usage: clustering.py [-h] [--pdbfiles PDBFILES] [--cpickle CPICKLE] [-i INPUT]
                     [-o OUTPUT] [-c CHECK] [-n NUMBER] [-s SIMILARITY]
                     [-s2 SIMILARITY_WITH_SIMILAR_LIGAND]
                     [-l LIGAND_SIMILARITY] [-d DATA_ROOT] [--posedir POSEDIR]
                     [--randomize RANDOMIZE] [-v] [--reduce REDUCE]

create train/test sets for cross-validation separating by sequence similarity
of protein targets and rdkit fingerprint similarity

optional arguments:
  -h, --help            show this help message and exit
  --pdbfiles PDBFILES   file with target names, paths to pbdfiles of targets,
                        paths to ligand smile (separated by space)
  --cpickle CPICKLE     cpickle file for precomputed distance matrix and
                        ligand similarity matrix
  -i INPUT, --input INPUT
                        input .types file to create folds from, it is assumed
                        receptors in pdb named directories
  -o OUTPUT, --output OUTPUT
                        output name for clustered folds
  -c CHECK, --check CHECK
                        input name for folds to check for similarity
  -n NUMBER, --number NUMBER
                        number of folds to create/check. default=3
  -s SIMILARITY, --similarity SIMILARITY
                        what percentage similarity to cluster by. default=0.5
  -s2 SIMILARITY_WITH_SIMILAR_LIGAND, --similarity_with_similar_ligand SIMILARITY_WITH_SIMILAR_LIGAND
                        what percentage similarity to cluster by when ligands
                        are similar default=0.3
  -l LIGAND_SIMILARITY, --ligand_similarity LIGAND_SIMILARITY
                        similarity threshold for ligands, default=0.9
  -d DATA_ROOT, --data_root DATA_ROOT
                        path to target dirs
  --posedir POSEDIR     subdir of target dirs where ligand poses are located
  --randomize RANDOMIZE
                        randomize inputs to get a different split, number is
                        random seed
  -v, --verbose         verbose output
  --reduce REDUCE       Fraction to sample by for reduced files. default=0.05
```
INPUT is a types file that you want to create clusters for

Either CPICKLE or PDBFILES needs to be input for the script to work.

PDBFILES is a file of target_name, path to pdbfile of target, and path to the ligand smile (separated by space)
CPICKLE is either the dump from running clustering.py one time, or the output from Case 2 (below) and allows you
to avoid recomputing the costly protein sequence and ligand similarity matrices needed for clustering.

When running with PDBFILES, the script will output PDBFILES.pickle which contains (distanceMatrix, target_names, ligansim),
where distanceMatrix is the matrix of cUTDM2 distance between the protein sequences,
target_names is the list of targets, and ligandsim is the matrix of ligand similarities.

When running with CPICKLE, only the new .types files will be output.

A typical usage case would be to create 5 different seeds of 5fold cross-validation.
First, we create seed0, which also will compute the matrices needed. This depends on having
INPUT a types file that we want to generate clusters for
PDBFILES (target_name path_to_pdb_file path_to_ligand_smile) for each target in types
```
clustering.py --pdbfiles my_info --input my_types.types --output my_types_cv_seed0_ --randomize 0 --number 5
```
Next we run the following four commands to generate the other 4 seeds
```
clustering.py --cpickle matrix.pickle --input my_types.types --output my_types_cv_seed1_ --randomize 1 --number 5
clustering.py --cpickle matrix.pickle --input my_types.types --output my_types_cv_seed2_ --randomize 2 --number 5
clustering.py --cpickle matrix.pickle --input my_types.types --output my_types_cv_seed3_ --randomize 3 --number 5
clustering.py --cpickle matrix.pickle --input my_types.types --output my_types_cv_seed4_ --randomize 4 --number 5
```

### Case 2: Running compute_*.py pipeline
First, we will use the compute_seqs.py to generate the needed input files
```
usage: compute_seqs.py [-h] --pdbfiles PDBFILES [--out OUT]

Output the needed input for compute_row. This takes the format of
"<target_name> <ligand smile> <target_sequence>" separated by spaces

optional arguments:
  -h, --help           show this help message and exit
  --pdbfiles PDBFILES  file with target names, paths to pbdfiles of targets,
                       and path to smiles file of ligand (separated by space)
  --out OUT            output file (default stdout)

```

PDBFILES is the same input that would be given to clustering.py.

For the rest of this pipeline, I will consider the output of compute_seqs.py to be comp_seq_out.

Second, we will run compute_row.py for each line in the output of compute_seqs.py
```
usage: compute_row.py [-h] --pdbseqs PDBSEQS -r ROW [--out OUT]

Compute a single row of a distance matrix and ligand similarity matrix from a
pdbinfo file.

optional arguments:
  -h, --help         show this help message and exit
  --pdbseqs PDBSEQS  file with target names, ligand smile, and sequences
                     (chains separated by space)
  -r ROW, --row ROW  row to compute
  --out OUT          output file (default stdout)

```
Here PDBSEQS is the output of compute_seqs.py. For example, to compute row zero
and store the output into the file row0:
```
compute_row.py --pdbseqs comp_seq_out --row 0 --out row0
```
For the next part, I assume that the output of compute_row.py is row[num] where [num] is
the row that was computed.

Third, we will run combine_rows.py to create the cpickle file needed for input into clustering.py
```
combine_rows.py row*
```
combine_rows.py accepts any number of input files, and outputs matrix.pickle

Lastly, we run clustering.py as follows
```
clustering.py --cpickle matrix.pickle --input my_types.types --output my_types_cv_
```
