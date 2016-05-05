class: center, middle

# Algorithms and their children

(What happens when you run an algorithm as child?)


---

## Questions

* Mantid: algorithms + workspaces

* Algorithms: managed/unmanaged, child:
  - What are they?
  - What are the differences? (visible and invisible)

* Wiki / doxygen / sources
  - Overall description?
  - Misleading issues (`Python != cpp`, workspace groups)

---

## Understanding algorithms

- Training course(s):

  - [Extending Mantid with Python](http://www.mantidproject.org/Extending_Mantid_With_Python)

  - Not much about children. From algorithms, run other algorithms through the "simple API". Example:

```python
from mantid.simpleapi import Load, Scale, DeleteWorkspace

_tmpws = Load(Filename=self.getPropertyValue("Filename"))
```

- [Writing an algorithm (C++)[http://www.mantidproject.org/Writing_an_Algorithm]

```cpp

Mantid::API::Algorithm_sptr childAlg = createChildAlgorithm("Segfault");
```

--
Additional documentation:
- [Python vs. C++ algorithms](http://www.mantidproject.org/Python_Vs_C%2B%2B_Algorithms)

- [Usage examples / docs-tests](http://www.mantidproject.org/Algorithm_Usage_Examples)

- [Multithreading](http://www.mantidproject.org/Multithreading_in_Mantid_Algorithms)

---

## Running an algorithm

- `IAlgorithm::execute()`

- Creating / running as child (from [Writing an algorithm (C++)](http://www.mantidproject.org/Writing_an_Algorithm):

```cpp
Mantid::API::Algorithm_sptr childAlg = createChildAlgorithm("AlgorithmName");

childAlg->setPropertyValue("number", 0);
childAlg->setProperty<Workspace_sptr>("Workspace",workspacePointer);
childAlg->execute();
```

- But also: `create()` + `setChild(true)`

- And, what is different in child algorithms?

---

## The many ways of creating an algorithm

`IAlgorithm` / `Algorithm` /  / `AlgorithmManager` / `AlgorithmProxy`?


```cpp

Algorithm_sptr alg =
    AlgorithmManager::Instance().createUnmanaged("DeleteWorkspace");
alg->initialize();
alg->setChild(true);
alg->setProperty("Workspace", tempWs);
alg->execute();
```

```python

rfi_alg = self.createChildAlgorithm(name='RawFileInfo', enableLogging=False)
rfi_alg.setProperty('Filename', run_str)
rfi_alg.execute()
nperiods = rfi_alg.getProperty('PeriodCount').value
```

- `createChildAlgorithm()` initializes the algorithm

- `setChild()` does not initialize?

---
## Create Unmanaged

- [`AlgorithmManager::createUnmanaged()`](https://github.com/mantidproject/mantid/blob/master/Framework/API/inc/MantidAPI/AlgorithmManager.h#L62-L64).

- What is an "unmanaged" algorithm?

- Used ~220 times, ~50 times in unit tests, sometimes in system tests.

```cpp
AlgorithmManager::create(const std::string &algName, const int &version = -1, bool makeProxy = true)

AlgorithmManager::createUnmanaged(const std::string &algName, const int &version = -1) const;
```

```python
cropped_name = dark_run_ws_name + "_cropped"
alg_crop = AlgorithmManager.createUnmanaged("CropWorkspace")
alg_crop.initialize()
alg_crop.setChild(True)
alg_crop.setProperty("InputWorkspace", dark_run_ws)
alg_crop.setProperty("OutputWorkspace", cropped_name)
alg_crop.setProperty("StartWorkspaceIndex", start_ws_index)
alg_crop.setProperty("EndWorkspaceIndex", end_ws_index)
alg_crop.execute()
dark_run_ws= alg_crop.getProperty("OutputWorkspace").value
```

- **All "createChildAlgorithm()" child algorithms are
    [unmanaged](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L751-L761)**

---

## The many ways of executing an algorithm

- `IAlgorithm::execute()`

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

- [IAlgorithm::executeAsChildAlg()](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L670-L687)

```cpp
IAlgorithm_sptr loadAlg;
loadAlg = createChildAlgorithm("HFIRLoad", 0.1, 0.3);
loadAlg->setProperty("Filename", fileName);
loadAlg->setProperty("ReductionProperties", reductionManagerName);
loadAlg->executeAsChildAlg();
```

---

## Still more ways of executing an algorithm

- `AlgorithmProxy::executeAsync()`

- You can also (ab)use `Algorithm::fromString()`
  (example in [AlgorithmTest.h](https://github.com/mantidproject/mantid/blob/master/Framework/API/test/AlgorithmTest.h#L316-L320))

  Meant to de-serialize algorithm objects, but in the past it has been
  seen used as an alternative "create".

- And if you're a `DataProcessorAlgorithm`:

```cpp
boost::shared_ptr<Algorithm> DataProcessorAlgorithm::createChildAlgorithm(
    const std::string &name, const double startProgress,
    const double endProgress, const bool enableLogging, const int &version)
```

( = `Algorithm::createChildAlgorithm(...)` + history)

---

## What is different in child algorithms?

- How output workspaces are produced by / can be retrieved from the algorithm

- Logging

- History in workspaces

- Progress bar/report

All this can be controlled with
[Algorithm::createChildAlgorithm](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L733-L799):

```cpp
virtual boost::shared_ptr<Algorithm> createChildAlgorithm(
    const std::string &name, const double startProgress = -1.,
    const double endProgress = -1., const bool enableLogging = true,
    const int &version = -1);
```
And `void enableHistoryRecordingForChild(const bool on)`

---

## Most important difference

- Child: **Output workspaces [are not put in the ADS (Analysis Data
  Service)](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L586-L591)**

- In algorithms you can use temporary variables with child algorithms:

```cpp
IAlgorithm_sptr duplicate = createChildAlgorithm("CloneWorkspace");
duplicate->initialize();
duplicate->setProperty<Workspace_sptr>(
    "InputWorkspace", boost::dynamic_pointer_cast<Workspace>(inputWs));

duplicate->execute();

Workspace_sptr temp = duplicate->getProperty("OutputWorkspace");
MatrixWorkspace_sptr outputWs =
    boost::dynamic_pointer_cast<MatrixWorkspace>(temp);
```

No need to `setPropertyValue("OutputWorkspace", "dummy")`


- A feature of (C++) parent/child algorithms that would be useful
  elsewhere (interfaces, tests, etc.): produce workspaces that are not
  stored in the ADS.

---

## Using child algorithms in unit tests and custom interfaces?

- Many tests do `setChild(true)`

```cpp
LoadNexusProcessed loadAlg;
loadAlg.setChild(true);
loadAlg.initialize();
loadAlg.setPropertyValue("Filename", "SingleCrystalPeakTable.nxs");
loadAlg.setPropertyValue("OutputWorkspace", "dummy");
loadAlg.execute();

Workspace_sptr ws = loadAlg.getProperty("OutputWorkspace");
auto peakWS =
    boost::dynamic_pointer_cast<Mantid::DataObjects::PeaksWorkspace>(ws);
```

Outside of algorithms: (some) logging will be hidden?, but you still need to name the output workspaces.


- In interfaces you can exploit the "hidden" names trick: "__" prefix.


---

## Other differences: progress and history

- Progress: `IAlgorithm::setChildStartProgress()` / `setChildEndProgress()`

- By default children are not included in the workspace history

- Can use `IAlgorithm::enableHistoryRecordingForChild(bool)` to change that

- Somewhere in Algorithm.cpp:

```cpp
bool Algorithm::trackingHistory() {
  return (!isChild() || m_recordHistoryForChild);
}
```

- History recording enabled in: `DataProcessorAlgorithm` and `Comment`


---
## Problems: Python algorithms behave differently!

- They [always put workspaces in the ADS (with a
  name)](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L153-L162):

```cpp
/** Do we ALWAYS store in the AnalysisDataService? This is set to true
 * for python algorithms' child algorithms
 *
 * @param doStore :: always store in ADS
 */
void Algorithm::setAlwaysStoreInADS(const bool doStore) {
  m_alwaysStoreInADS = doStore;
}
```

In [simple
API](https://github.com/mantidproject/mantid/blob/master/Framework/PythonInterface/mantid/simpleapi.py#L925-L931):

```python
    if parent is not None:
        alg = parent.createChildAlgorithm(name, version)
        alg.setLogging(parent.isLogging()) # default is to log if parent is logging

        # Historic: simpleapi functions always put stuff in the ADS
        #           If we change this we culd potentially break many users' algorithms
        alg.setAlwaysStoreInADS(True)
```

--
- Also log normally. Then what does `setChild()` on a Python algorithm do?


---
## Problems? Workspaces, the ADS and names.

- To circumvent issues?
  [ScopedWorkspace](https://github.com/mantidproject/mantid/blob/master/Framework/API/inc/MantidAPI/ScopedWorkspace.h):
  "hidden" name ("__" trick) generated with a random sequence
  number. Used in:

  - Some 6 `Algorithms` and `DataHandling` tests
    (SaveNexusProcessedTest, LoadMuonNexus1Test, RebinTest,
    ChangeTimeZeroTest, etc.)

  - Muons: MuonAnalysis interface, PlotAsymmetryByLogValue algorithm

--

- **When the output workspace is a `WorkspaceGroup` you'll need to give it a name**




---



---

## Child algorithms that become non-child

In `Algorithm::processGroups()`, [created as child but then deprived
of its
childness](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L1244-L1249):

```cpp
Algorithm_sptr alg_sptr = this->createChildAlgorithm(
    this->name(), -1, -1, this->isLogging(), this->version());
// Don't make the new algorithm a child so that it's workspaces are stored
// correctly
alg_sptr->setChild(false);
```


---

## Internals: Managed/Unmanaged/Child/Parent?

Managed algorithms have a non-zero ID. Unamanged algorithms have ID=0

[//]: <> (This comes from IAlgorithm.h)


```cpp
  /** To set whether algorithm is a child.
   *  @param isChild :: True - the algorithm is a child algorithm.  False - this
   * is a full managed algorithm.
   */
  virtual void setChild(const bool isChild) = 0;
```

Children: workspaces not locked/unlocked in `Algorithm::execute()`


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

- Then what does setChild on a Python algorithm do? (Note: there is IAlgorithm::setLogging() / isLogging())

Still, in C++,

For `WorkspaceGroup` output properties,
[Algorithm::createChildAlgorithm()](https://github.com/mantidproject/mantid/blob/master/Framework/API/src/Algorithm.cpp#L770-L780)
exceptionally uses
[Property::createTemporaryValue()](https://github.com/mantidproject/mantid/blob/master/Framework/Kernel/src/Property.cpp#L148-L155)
