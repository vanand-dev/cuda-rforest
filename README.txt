GUIDE for RandomForest project (a random forest model implemented in CUDA and C)
  Source: rforest.cpp - The RandomForest class; an instance of it represents a
  random forest model that takes a matrix/dataset as input, which it can train
  on and use for predictions.

    main.cpp - Tests the model by creating and using instances of the
    RandomForest class, as described further below.

    rforest.cu - Contains the main kernel that computes losses given data over
    every possible split in the data (parallelism over # of points and # of
    features)

    utils.cpp - Contains functions that act on nodes and i/o functions to print
    structures or errors and read in data

    utils.hpp - Contains the definition of a node struct that is used to
    construct the decision trees in a random forest; also exposes functions in
    utils.cpp

    constants.hpp - Contains constants used as defaults for parameterizing the
    random forest. These can be overridden by passing arguments through the
    commandline.

  Benchmarking: times.txt - Contains times for training trees (forests of 1
    tree) and forests on various dataset shapes and model parameters. The train
    time was relatively consistent between runs, so a single trial was used for
    most measurements as an estimate.

  Git: https://github.com/vanand-dev/cuda-rforest Contains versioning for the
    latter half of the project after CPU version was working and partial CUDA
    version was implemented (it was useful for branching into different designs
    and documenting progress).


IMPORTANT NOTES:
  1. About random forests "A decision tree is a graph that uses a branching
     method to illustrate every possible outcome of a decision." When
     constructed/trained on data, they are serve as a trained model to predict
     outcomes on other data. Bagging is a method of aggregating predictions from
     a number of decision trees that are trained on a subset of the data, called
     "subsampling" and referred to in the code as such. By subsampling, we
     reduce model complexity/variance and by aggregating many trees we reduce
     bias. In this code, aggregating predictions is done by taking their mode
     (majority voting).

  3. What the software does It creates two instances of random forests serially.
     One is set to only use the CPU and the other uses the GPU (CUDA and
     CUBLAS). The main differences occur between the two modes in building the
     tree, measuring loss, and an intermediate matrix-transposition step for
     making predictions.

  4. Argument passing/options The executable requires a path to a dataset. A
     default one was created by the python3 script ./make_data.py and exists in
     ./data/data.csv. All other optional arguments are denoted below in braces
     [-o/--option <argument>]. Usage: ./rforest <path of csv> [-s/--shape <#
     points> <# features>] [-t/--trees <# of trees>] [-d/--depth <max depth of
     trees>] [-f/--frac <fraction to use for subsampling, if <=0, no sampling]
     [-v/--verbose <level of verbosity, default 1>] If the optional arguments
     are not set, these values are set to their defaults in constants.hpp. The
     function of these arguments are also describe in detail in constants.hpp.

  5. What is called for each forest As shown in main.cpp, forest.run(),
     forest.predict() and forest.get_mse_loss() are mainly called. This fits the
     model according to data and then predicts on the training data using the
     model. It does this while printing times for computation as well as the
     forest itself (depending on verbosity).

  6. Building the model The GPU version will use the kernel in rforest.cu to
     find the next optimal split in the data at each node in the tree (when the
     number of points is below a threshold, it will switch to using only the
     CPU, however). The CPU version will do this using members of the
     RandomForest class. In both versions, the tree is constructed in
     RandomForest::node_split() by solving each of the tree's nodes in a depth-
     first recursive manner . Although the CUDA kernel developed to find the
     optimal split at each node in a decision tree is less than a hundred lines,
     it went through many design-changes and refactoring for both optimizations
     and correctness. The analogue of the CPU functions to the kernel are in
     data_split() paired with get_info_loss(). The code was carefully commented,
     so please see code for further implementation details. To learn more about
     the model, start with rforest.hpp and then rforest.cpp.

  7. Hardware assumptions This software was tested on a GTX 1060 with 1024
     maximum threads per block, 48kb of shared memory per block, and 6.1 CUDA
     Capability. The software will use "THREADS_PER_BLOCK" threads per block
     defined in rforest.cuh (set to 1024). It will allocate 16*THREADS_PER_BLOCK
     bytes of shared memory. Therefore, it will use 16kb of shared memory per
     block.

  8. Testing correctness This was confirmed for both the CPU and GPU modes by
      A. The training loss should always, consistently be 0 when there is no
         subsampling or depth limit of the decision tree.
      B. The Python, C, and CUDA versions, although referencing each other, were
         implemented significantly differently. The Python version was checked
         against sklearn's random forest in terms of MSE loss, visually, and
         manually checking on small datasets of about 10 points. The C and CUDA
         versions were checked against the Python version and then against each
         other.