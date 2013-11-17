# Exelixi

*Exelixi* is a distributed framework for running [genetic algorithms] at scale.
The framework is based on [Apache Mesos] and the code is mostly implemented in Python.

On the one hand, this project provides a tutorial that shows how to build distributed frameworks in [Apache Mesos].
On the other hand, this implements a general-purpose [GA] platform that emphasizes scalability and fault tolerance,
while leveraging the wealth of available Python analytics packages.


## Background
### Component Definitions

_Individual_:
state is represented by a feature set and a fitness value obtained by applying a fitness function to that feature set.

_Population_:
a collection of Individuals, which breed other Individuals.

_Fossil Record_:
an archive of Individuals which did not survive, persisted to durable storage and used to limit ergodic behaviors.


## Implementation
### Design For Scalability

To implement a [GA] in Exelixi, simply extend two classes in Python.
First, subclass the _Individual_ class to customize the following operations:
* randomly generate a feature set
* handle codex for serializing a feature set
* mutate a feature set
* breed a pair of parents to produce a child
* calculate (or approximate) a fitness function

Individuals get represented as key/value pairs.
The value consists of a tuple (fitness value, generation) and the key is constructed from a feature set. 

To construct a key, a feature set is expressed as an JSON chunk serialized by being compressed and converted into hexadecimal ASCII armor.
This ASCII string is then split into N-character chunks.
These chunks define a path in [HDFS] for persisting the Individual in the fossil record.

Let's consider how to store an Individual in [HDFS], given some UUID as a job's unique prefix (e.g., "FE92A") and a specific key (e.g., "E45F", "BC19", "234D"), plus a fitness value (e.g., 0.5654) and generation number (e.g., 231)
This particular Individual would be represented as the pair:

    hdfs://FE92A/E45F/BC19/234D, [0.5654, 231]

Note that feature set serialization (i.e., construction of a key) and fitness function calculation only need to be performed once.
This allows for idempotence in the overall data collection.
e.g., append-only updates to [HDFS], which can be used to reconstruct state following a node or process failure.


### Framework

The _framework_ is a long-running process that:
* maintains _operational state_ (e.g., system parameters) in [Zookeeper]
  * Python classes for customization
  * [HDFS] directory prefix
  * n_exe
  * list of executor endpoints from [Marathon]
* maintains _logical state_ (e.g., model parameters) in [Zookeeper]:
  * n_pop
  * n_gen
  * current_gen
  * retention_rate
  * selection_rate
  * mutation_rate
* generates the [HDFS] directory prefix
* initializes the pool of executors
* iterates through the phases of each generation (selection/mutation, breeding, evaluation, reporting, shuffle)
* restores state for itself or for any executor after a failure
* reports results at any point -- including final results after the generations have completed


### Executor

An _executor_ is a service running on a Mesos slave that:
* implements a simple cache backed by [HDFS]
* provides a lookup service for the feature space vs. fitness of known attempts
* generates a pool of "live" Individuals at initialization or recovery
* maintains "live" Individuals in memory
* persists serialized Individuals to durable storage
* calculates a partial histogram for the distribution of fitness
* shuffles the local population among neighboring executors
* applies a filter to "live" Individuals to select parents for the next generation
* handles mutation, breeding, and evaluation of "live" Individuals


[Apache Mesos]: http://mesos.apache.org/ "Apache Mesos"
[GA]: http://en.wikipedia.org/wiki/Genetic_algorithm "Genetic algorithms"
[HDFS]: http://hadoop.apache.org/ "HDFS"
[Marathon]: https://github.com/mesosphere/marathon "Marathon"
[Zookeeper]: http://zookeeper.apache.org/ "Apache Zookeeper"
[genetic algorithms]: http://en.wikipedia.org/wiki/Genetic_algorithm "Genetic algorithms"
