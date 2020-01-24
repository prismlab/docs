##Benchmarking Irmin##

####Rough Draft : Synthetic Benchmarks for Irmin####

* Benchmarking 2 text blobs.
  * Take 2 text blobs
  * Add one of the text blobs to the master branch, let's call it *log_one*
  * Take the second blob and create a chunk of that blob into a list of strings.
  * The Iterative part :
    * Clone *log_one*
    * Add one element from the list of strings
    * merge that branch into master
    * repeat this until all the elements of the string go through the same process.
  * End

* Benchmarking the merge function
  * To benchmark the function go through the [blogpost](https://github.com/prismlab/docs/wiki/Adding-a-benchmark-to-Sandmark) created in the prismlab github repo.
