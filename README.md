# monocle-docker
Environment for executing the monocle trajectory inference algorithm.
Contains [functions](https://github.com/Stuartlab-UCSC/traj-converters) to convert the output to [common formats](https://github.com/Stuartlab-UCSC/traj-formats). The container is hosted on [docker hub](https://hub.docker.com/r/stuartlab/monocle/).


This readme outlines three ways the container can be used. You'll need docker installed to follow.

* [Execute the container's native analysis script.](#min)
* [Develop analysis inside container using Rstudio.](#container)
* [Execute a user defined script inside the image.](#already)


For instructions below I suggest creating a shared volume,
`./shared` in the directory you are launching the docker from. This 
volume acts as shared storage. The shared storage is the easiest way to
access input and output from the container.

You can run this command in bash (R needs to be installed) to create the test expression data, exp.tab:

`R -e 'set.seed(13);write.table(replicate(110, rnbinom(500, c(3, 10, 45, 100), .1)),file="shared/exp.tab",col.names=1:110, sep="\t")'`

## <a name="min"></a>Execute the container's native analysis script:
We are trying to establish a convention where containers for TI algorithms have a`run_method` script in the image's $PATH. This allows a uniform interface for containers with different algorithms. The pattern is:

`docker run -v $(pwd):/data <container_name> run_method <tab delim exp file> <output.json>`

If the second positional argument is ommitted then a file called output_*method*.json will be produced. Hence, if your expression file 'exp.tab' is in your current working directory:

`docker run -v $(pwd):/data stuartlab/monocle run_method exp.tab`

will produce a 'output_monocle.json' file.

## <a name="container"></a>Develop inside the container with Rstudio:

Move into the directory you'd like to do your work in, and create a `shared` directory for the docker container.

`cd some/directory && mkdir shared`

Let's say you have a tab delimited expression file, `exp.tab`, you'd like to run TI analysis on. Move it into the shared directory.

`cp ../../place/your/data/is/exp.tab shared`

If you don't have an exp.tab file, you may create example data with R.

`R -e 'set.seed(13);write.table(replicate(110, rnbinom(500, c(3, 10, 45, 100), .1)),file="shared/exp.tab",col.names=1:110, sep="\t")'`

Then pull the image down (if it doesn't exist on your machine), and run an Rstudio session in your browser.

`docker run -v $(pwd)/shared:/home/shared -d -p 8787:8787 -e PASSWORD=ABC123 -e ROOT=TRUE stuartlab/monocle`

From there open your favorite browser (tested on chrome) at `http://localhost:8787/`. The password is `ABC123` and username is `rstudio`.

Now you can use Rstudio to create a script that runs on your data, and save
the script to `/home/shared` inside the container. An example
script could be something like:

```R
library(monocle)
source("/home/traj-converters/src/R/monocle_convert.r")

read_data <- function(){
  # Read tab matrix from shared dir.
  as.matrix(read.table("/home/shared/exp.tab", sep="\t", header=T))
}
monocle_analysis <- function(expression_matrix){
  # Return a completed monocle object
  cds <- newCellDataSet(expression_matrix)
  cds <- estimateSizeFactors(cds)
  cds <- estimateDispersions(cds)
  ordering_genes <- subset(dispersionTable(cds), mean_expression>=0.1)
  cds <- setOrderingFilter(cds, ordering_genes)
  cds <- reduceDimension(cds, max_components = 2, method = "DDRTree")
  cds <- orderCells(cds)
  return(cds)
}

# Execute pipeline.
exp_mat <- read_data()
monocle_obj <- monocle_analysis(exp_mat)

# Write to common format.
write_common_json(monocle_obj, file="/home/shared/monocle.json")
write_cell_x_branch(monocle_obj, file="/home/shared/monocle.cxb.tab")
```

In reality, you'll want to toy with the monocle_analysis function and investigate the resulting monocle object with the various plot functions provided by monocle. Once you're happy with the analysis and you've saved the script in the container, say `/home/shared/my_analysis_script.r`, you can [run the script in the container](#already) to reproduce the analysis.

## <a name="already"></a>Execute a user defined script inside the image:
This command assumes that you have a script `my_analysis_script.r` in a directory named `shared` in your current working directory. To make your life easier I suggest developing this script inside the container as [above](#container). If you did not do that, it's necessary that the paths in `my_analysis_script.r` match the container's file system. This amounts to changing the paths that read and write data from `path/on/machine/*` to `/home/shared/*`. 

Make sure you have your data and script in a directory named `shared` so the docker has access to them.

`mkdir shared && cp ../../data/for/script ./shared && cp ../../my_analysis_script ./shared`

Then bind the shared directory and execute the script.

`docker run -v $(pwd)/shared:/home/shared stuartlab/monocle Rscript /home/shared/my_analysis_script.r`
