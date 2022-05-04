# Map Reduce

1. few rules
  The map phase should divide the intermediate keys into buckets for nReduce reduce tasks, where nReduce is the argument that main/mrmaster.go passes to MakeMaster().

  The worker implementation should put the output of the X'th reduce task in the file mr-out-X.

  A mr-out-X file should contain one line per Reduce function output. The line should be generated with the Go "%v %v" format, called with the key and value. **Have a look in main/mrsequential.go for the line commented "this is the correct format".** The test script will fail if your implementation deviates too much from this format.

  You can modify mr/worker.go, mr/master.go, and mr/rpc.go. You can temporarily modify other files for testing, but make sure your code works with the original versions; we'll test with the original versions.

  The worker should put intermediate Map output in files in the current directory, where your worker can later read them as input to Reduce tasks.

  main/mrmaster.go expects mr/master.go to implement a Done() method that returns true when the MapReduce job is completely finished; at that point, mrmaster.go will exit.

  When the job is completely finished, the worker processes should exit. A simple way to implement this is to use the return value from call(): if the worker fails to contact the master, it can assume that the master has exited because the job is done, and so the worker can terminate too. Depending on your design, you might also find it helpful to have a "please exit" pseudo-task that the master can give to workers.

2. hints
  One way to get started is to modify mr/worker.go's Worker() to send an RPC to the master asking for a task. Then modify the master to respond with the file name of an as-yet-unstarted map task. Then modify the worker to read that file and call the application Map function, as in mrsequential.go.

  The application Map and Reduce functions are loaded at run-time using the Go plugin package, from files whose names end in .so.

  If you change anything in the mr/ directory, you will probably have to re-build any MapReduce plugins you use, with something like go build -buildmode=plugin ../mrapps/wc.go
  This lab relies on the workers sharing a file system. That's straightforward when all workers run on the same machine, but would require a global filesystem like GFS if the workers ran on different machines.

  A reasonable naming convention for intermediate files is mr-X-Y, where X is the Map task number, and Y is the reduce task number.

  The worker's map task code will need a way to store intermediate key/value pairs in files in a way that can be correctly read back during reduce tasks. One possibility is to use Go's encoding/json package. To write key/value pairs to a JSON file:
  ```go
    enc := json.NewEncoder(file)
    for _, kv := ... {
      err := enc.Encode(&kv)
  ```
  and read such file back
  ```go
    dec := json.NewDecoder(file)
    for {
      var kv KeyValue
      if err := dec.Decode(&kv); err != nil {
        break
      }
      kva = append(kva, kv)
    }
  ```
  The map part of your worker can use the ihash(key) function (in worker.go) to pick the reduce task for a given key.

  You can steal some code from mrsequential.go for reading Map input files, for sorting intermedate key/value pairs between the Map and Reduce, and for storing Reduce output in files.

  The master, as an RPC server, will be concurrent; don't forget to lock shared data.

  Use Go's race detector, with go build -race and go run -race. test-mr.sh has a comment that shows you how to enable the race detector for the tests.

  Workers will sometimes need to wait, e.g. reduces can't start until the last map has finished. One possibility is for workers to periodically ask the master for work, sleeping with time.Sleep() between each request. Another possibility is for the relevant RPC handler in the master to have a loop that waits, either with time.Sleep() or sync.Cond. Go runs the handler for each RPC in its own thread, so the fact that one handler is waiting won't prevent the master from processing other RPCs.

  The master can't reliably distinguish between crashed workers, workers that are alive but have stalled for some reason, and workers that are executing but too slowly to be useful. The best you can do is have the master wait for some amount of time, and then give up and re-issue the task to a different worker. For this lab, have the master wait for ten seconds; after that the master should assume the worker has died (of course, it might not have).

  To test crash recovery, you can use the mrapps/crash.go application plugin. It randomly exits in the Map and Reduce functions.

  To ensure that nobody observes partially written files in the presence of crashes, the MapReduce paper mentions the trick of using a temporary file and atomically renaming it once it is completely written. You can use ioutil.TempFile to create a temporary file and os.Rename to atomically rename it.

  test-mr.sh runs all the processes in the sub-directory mr-tmp, so if something goes wrong and you want to look at intermediate or output files, look there.