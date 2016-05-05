class: center, middle

# Algorithms and their children

(What happens when you run an algorithm as child, unmanaged, etc.?)


---

## Questions

* Managed/unmanaged, child:
  - What is the point?
  - What are the consequences? (visible and invisible)

* Wiki / doxygen / sources
  - Overall description?
  - Misleading issues (workspace groups, `Python != cpp`)

---

## Understanding algorithms



training course(s)  *** TODO ***

- [Extending Mantid with Python](http://www.mantidproject.org/Extending_Mantid_With_Python)

(From algorithms, run other algorithm through the "simple API":
` Load(...) `
)



- [Writing an algorithm (C++)[http://www.mantidproject.org/Writing_an_Algorithm]

```cpp

Mantid::API::Algorithm_sptr childAlg = createChildAlgorithm("AlgorithmName");
```


Other pages:
- [Python vs. C++ algorithms](http://www.mantidproject.org/Python_Vs_C%2B%2B_Algorithms)

- [Usage examples / docs-tests](http://www.mantidproject.org/Algorithm_Usage_Examples)

- [Multithreading](http://www.mantidproject.org/Multithreading_in_Mantid_Algorithms)
---

## Running an algorithm is straightforward.

- `IAlgorithm::execute()`

- Creating / running as child (from [Writing an algorithm (C++)[http://www.mantidproject.org/Writing_an_Algorithm]):

```
Mantid::API::Algorithm_sptr childAlg = createChildAlgorithm("AlgorithmName");

childAlg->setPropertyValue("number", 0);
hildAlg->setProperty<Workspace_sptr>("Workspace",workspacePointer);
childAlg->execute();
```

- But also: create / setChild

- But, what is different in child algorithms?

--

- The many ways

Algorithm / IAlgorithm / AlgorithmManager / AlgorithmProxy?


```cpp


```

```python

```

- createChild initialises
- setChild does not initialize?


---

## The more many ways

- `IAlgorithm::execute()`

- `IAlgorithm::executeAsChildAlg()`

- `IAlgorithm::setChild()`

```cpp
    SaveNexusProcessed alg;
    alg.setRethrows(true);
    alg.setChild(true);
    alg.initialize();
    alg.setProperty("Filename", output_filename);
    // ... alg.setProperty("InputWorkspace", group_ws);
    alg.execute();
    // ...
```

---
## More ways to execute an algorithm

- `AlgorithmProxy::executeAsync()`

- You can also (ab)use `Algorithm::fromString()`
  (example in [AlgorithmTest.h](https://))

  Meant to de-serialize algorithm objects, but has been seen used as an
  alternative "create".

- And if you're a `DataProcessorAlgorithm`:

```cpp
boost::shared_ptr<Algorithm> DataProcessorAlgorithm::createChildAlgorithm(
    const std::string &name, const double startProgress,
    const double endProgress, const bool enableLogging, const int &version)
```

---
## Create Unmanaged


- `AlgorithmManager::createUnmanaged()`

- What is an "unmanaged" algorithm:
  [`AlgorithmManager::createUnmanaged()`](https://github.com/mantidproject/mantid/blob/master/Framework/API/inc/MantidAPI/AlgorithmManager.h#L62-L64)

- Used ~221 times, ~50 times in unit tests, sometimes in system tests.

AlgorithmManager::create(const std::string &algName, const int &version = -1, bool makeProxy = true)
AlgorithmManager::createUnmanaged(const std::string &algName, const int &version = -1) const;


From scripts/SANS/isis_reduction_steps.py
```python
alg_crop = AlgorithmManager.createUnmanaged("CropWorkspace")
```

- **All child algorithms are
    [unmanaged](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L751-L761)**

---

## What is different in child algorithms?

- How output workspaces are produced by / can be retrieved from the algorithm

- Logging

- History in workspaces

- Progress bar/report

All this can be controlled with [this method of Algorithm]():

```cpp
virtual boost::shared_ptr<Algorithm> createChildAlgorithm(
    const std::string &name, const double startProgress = -1.,
    const double endProgress = -1., const bool enableLogging = true,
    const int &version = -1);
```
 + `void enableHistoryRecordingForChild(const bool on)`

---

## Managed/Unmanaged/Child/Parent?


```cpp
  /** To set whether algorithm is a child.
   *  @param isChild :: True - the algorithm is a child algorithm.  False - this
   * is a full managed algorithm.
   */
  virtual void setChild(const bool isChild) = 0;
```


---

## Consequences


Managed algorithms have a non-zero ID. Unamanged algorithms have ID=0

[//]: <> (This comes from IAlgorithm.h)

---

## More consequences

```cpp
alg->setPropertyValue("OutputWorkspace", "dummy");
// ...
IMDHistoWorkspace_sptr visualHistoWs =
          alg->getProperty("OutputWorkspace");
```

```cpp
LoadNexusProcessed loadAlg;
loadAlg.setChild(true);
loadAlg.initialize();
// ...
Workspace_sptr ws = loadAlg.getProperty("OutputWorkspace");
auto peakWS =
     boost::dynamic_pointer_cast<Mantid::DataObjects::PeaksWorkspace>(ws);
```

Outside of algorithms: logging will be hidden, but you still need to name the output workspaces.
(trick: "__" prefix).


---

## Using algorithms in unit tests and custom interfaces



- many tests do `setChild(true)`



---

## Uses

If you run an alg using the "simple API", it is definitely not a child

---

## Consequences: progress

IAlgorithm::setChildStartProgress() / setChildEndProgress()



---

## Consequences: history


- By default children are not included in the workspace history

- Can use `IAlgorithm::enableHistoryRecordingForChild(bool)` to change that

- Somewhere in Algorithm.cpp:

```cpp
bool Algorithm::trackingHistory() {
  return (!isChild() || m_recordHistoryForChild);
}
```

- Enabled in: `DataProcessorAlgorithm` and `Comment`

---

## Problems?


When the output workspace is a `WorkspaceGroup` you'll need to give it a name.


--
## Workspaces

- `ScopedWorkspace`: "hidden" name generated with a random sequence number. Used in:
  - Some 6 `Algorithms` and `DataHandling` tests (SaveNexusProcessedTest, LoadMuonNexus1Test, RebinTest, ChangeTimeZeroTest, etc.)
  - Muons: MuonAnalysis interface, PlotAsymmetryByLogValue algorithm


---
## Internals:

/** Do we ALWAYS store in the AnalysisDataService? This is set to true
 * for python algorithms' child algorithms
 *
 * @param doStore :: always store in ADS
 */
void Algorithm::setAlwaysStoreInADS(const bool doStore) {
  m_alwaysStoreInADS = doStore;
}

TODO: link to comment in simpleapi.py
```python
    if parent is not None:
        alg = parent.createChildAlgorithm(name, version)
        alg.setLogging(parent.isLogging()) # default is to log if parent is logging

        # Historic: simpleapi functions always put stuff in the ADS
        #           If we change this we culd potentially break many users' algorithms
        alg.setAlwaysStoreInADS(True)
```


Note: `createChildAlgorithm()` also does `initialize()`.

---
## Internals - logging

In `Algorithm::execute()` you'll find:

```cpp
// no logging of input if a child algorithm (except for python child algos)
if (!m_isChildAlgorithm || m_alwaysStoreInADS)
  logAlgorithmInfo();
```


```cpp
// Put any output workspaces into the AnalysisDataService - if this is not
// a child algorithm
if (!isChild() || m_alwaysStoreInADS)
  this->store();
```

Python algorithms => m_alwaysStoreInADS==true

- For historic reasons.

- Then what does setChild on a Python algorithm do?


(Note there is: IAlgorithm::setLogging() / isLogging())



Still, in C++,

For `WorkspaceGroup` output properties,
`[Algorithm::createChildAlgorithm()](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L770-L780)`
exceptionally uses
[`Property::createTemporaryValue()`](https://github.com/mantidproject/mantid/blob/master/Framework/Kernel/src/Property.cpp#L148-L155)

---
## Internals

child: relevant for initialization

children: workspaces not locked/unlocked in `Algorithm::execute()`

---
## Internals


IAlgorithm::setAlwaysStoreInADS()

IAlgorithm::setLogging() / isLogging()


From Framework/API/test/AlgorithmTest.h:

```cpp
void testIsChild() {
  TS_ASSERT_EQUALS(false, alg.isChild());
  alg.setChild(true);
  TS_ASSERT_EQUALS(true, alg.isChild());
  alg.setChild(false);
  TS_ASSERT_EQUALS(false, alg.isChild());
}
                        

```