# Fibonacci Restful Service

Fibonacci restful service is designed to be high-scaliablity service to quickly calculate Fibonacci. 

### Fibonacci Algorithm
Matrix exponentiation (fast)
The algorithm is based on this innocent-looking identity (which can be proven by mathematical induction):
```
[1 1]n  [F(n+1) F(n)  ]
[1 0]  =[F(n)   F(n−1)].
```
It is important to use exponentiation by squaring with this algorithm, because otherwise it degenerates into the dynamic programming algorithm. This algorithm takes O(1) space and O(logn) operations. (Note: This is counted in terms of the number of bigint arithmetic operations, not primitive fixed-width operations.)

So, that gives me a idea that the calculation could be sperated into smaller jobs to finish together.
##### Related codes:
 * [FibonacciCoreCalucaltion](https://github.com/iamtangram/FibonacciRest/blob/master/src/main/java/com/emc/test/fibonacci/FibonacciCoreCalucaltion.java)

### Infrastructure Design
```
             +-------------------------------------------------------+
  Request    |                                                       |
             |                                                       |
 +------------->                                                     |
             |       HAProxy (Load Balance)                          |
             +-----------+--------------------------------+----------+
                         |  Compression                   |
                         |                                |
              +----------v----------+      +--------------v---------+
              |                     |      |                        |
              |                     |      |                        |
              |                     |      |                        +---------------------+
              |   spring container 1|...   |   spring container N   |                     |
              |                     |      |                        |                     |
              +-+-------------+-----+      +---------------+--------+                     |
                |             |                            |                              |
+-------------------------------------------------------------------------+  +------------v---------------+
|               |             |                            |              |  |                            |
|        +------v-----+ +-----v-----+               +------v---+     Heart Beat  +------------------+     |
|        |            | |           |               |          <-----------------> Worker Queue     |     |
|        |            | |           |               |          <----------------->                  |     |
|        |  JVM1(Docker)| JVM2 (Docker)             |   JVM-N  |          |  |   +------------------+     |
|        |            | |           |               |          |   start new one +------------------+     |
|        |            | |           |               |          |    <-----+--+---+  AutoScaling     |     |
|        +-^---------^+ +-----------+               +---^------+          |  |   |                  |     |
|          |         |                                  |                 |  |   +------------------+     |
|          |         |  Worker  Pool                    |                 |  |         Worker Monitor     |
+----------+---------+----------------------------------------------------+  +----------------------------+
          Map & Reduce                                  |
           +         |                                  |
           |         |                                  |
    +------v-+      +v--------+      +---------+   +----v----+
    |        +      |         |      |         |   |         |
    | Job1 (Thread) | JobN (Thread)  |  Job    |   |  Job    |
    |        |      |         |      |         |   |         |
    +--------+      +---------+      +---------+   +---------+


```
##### Notice
* Load balance is not designed yet for this test. I would continously upgrade and improve project codes on that.
* Each request comes from the load balance mechnisum and go through ***Spring container***. 
* The restful services will check the number first and then once the validation is passed, then it will trigger a JVM (or docker). See [code](https://github.com/iamtangram/FibonacciRest/blob/master/src/main/java/com/emc/test/rest/FibonacciCalculationResource.java).
* According to the max threadhold defined in the properties, JVM will start the counts of calculation jobs. The result of each jobs will be persisted to temp files. After all jobs are done, spring container will collect these in order and then output as ***compressed*** stream. The temp files will be ***deleted*** async.  See [Job Thread](https://github.com/iamtangram/FibonacciRest/blob/master/src/main/java/com/emc/test/fibonacci/FibonacciPartThread.java) and [Process](https://github.com/iamtangram/FibonacciRest/blob/master/src/main/java/com/emc/test/process/ProcessRunner.java). JVM will have a max maxheapsize (512m) for each run. So it prevent more JVM causing spring main process down.
* Properties is defined in [HERE](https://github.com/iamtangram/FibonacciRest/blob/master/src/main/resources/application.properties)
* When the request has a big and time-consuming task, then it will ***enqueue*** worker to worker queue and the status is ***BUSY*** meaning it is running. Suppose while the working is running and another big task comes in, then enqueue worker into queue. The queue could know if it is suitable to extend to run in another machine. If yes, then queue will record this machine id and automatically starts another container to run JVM. If no, so mark its status as ***ENQUEUE*** meaning it is waiting for previous big task done. When the first task is done, then status is marked as ***COMPLETE***.If the task has problem and not finish, then the status is marked as ***ABORTED***. It is in second version I would post. The diagram above is what I am thinking to do.

### How to Run

You need install maven 3 before running:

```sh
$ mvn spring-boot:run
```

Then you can GET http://localhost:9000/v1/rest/fibonacci/{id}

