## Benchmarking Irmin ##

#### Rough Draft : Synthetic Benchmarks for Irmin ####

* Benchmarking 2 text blobs.
  * Take 2 text blobs
  * Add one of the text blobs to the master branch, let's call it *log_one*
  * Take the second blob and create a chunk of that blob into a list of strings.
  * The 1st Iterative part (Cloning and Updating) :
    * Clone *log_one*
    * Add one element from the list of strings
    * repeat this until all the elements of the string go through the same process.
  * The 2nd Iterative part (Merging) :
    * All the updated branches are now closed to the master branch
  * End

* Benchmarking the merge function
  * To benchmark the function go through the [blogpost](https://github.com/prismlab/docs/wiki/Adding-a-benchmark-to-Sandmark) created in the prismlab github repo.
