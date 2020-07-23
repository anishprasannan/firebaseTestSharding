# Test sharding in firebase using past build results.
## Story so far.

Every Android app shop would have gone through a phase in their growth where they went from no UI automation tests, to our tests are timing out in firebase (We got too many tests... Argh, but we need more tests!). When this happened to us first, the immediate solution was to split the tests into suites and run them in parallel. A lot more tests later, we realized that manual suite splitting is not fun.

## What do we do ?

The solution was simple, google it. We saw a few different approaches. But, either they were too hard to implement in our CI env (Jenkins with groovy DSL pipelines) or did not suit our goals as an automated solution. Our idea was to split tests into suites, in our gradle build script.

## Idea.

After a bit of inspiration and brain storming the solution was to use test results (timing info) from a recent build, to split our suites evenly and run them in parallel in multiple devices so that they don't timeout on a device and, finish roughly at the same time.

## The important bits

The below stage in the pipeline sets up the test suites based on results from a previous successful test and runs them in parallel.

```
stage('Run Firebase Tests') {
                dir('YourDirectory') {
                    suitePath = "**/*Test.kt"
                    avoids = ["AvoidThisTest.kt","AvoidThisOneTooTest.kt"]
                    automattionType = "Name your test suites"
                    ultimateSuite = ultimateSuiteBuilder()
                    classNameCollections = testCaseBuilder(ultimateSuite, suitePath, avoids)

                    def stepsForParallel = classNameCollections.collectEntries {
                        ["TestSuite ${it.key}" : transformIntoStep(automattionType, it.key, it.value)]
                    }

                    parallel stepsForParallel
                }

            }
```

`avoids` let you avoid setup classes or other test classes from running.
`suitePath` can be made more absolute just to point out where your test classes are located.
I think most of this code is self explanatory, but will go through these methods below. `ultimateSuiteBuilder()` , `testCaseBuilder()` , `transformIntoStep()`

Constants that we will be using are listed below.

```
firebaseResultsBucket // Firebase bucket where, the results will be stored.
firebaseResultsBucketDir // Firebase directory where test results are stored.
firebaseLocalResultsDir // Local directory where test results are stored.
firebaseSuiteBuilderRemoteSync // Firebase sync folder for successful previous firebase results.
firebaseLocalSuiteBuilderSync // Local sync folder for successful previous firebase results.
```

### Collecting timing data `ultimateSuiteBuilder`

As the first step we need to find out execution times for test classes from a previous successful execution. For this we have setup a location in our `firebaseResultsBucket` called `firebaseSuiteBuilderRemoteSync` where we sync xml test result files of successful builds. 

The below function gets the xml result files from `firebaseSuiteBuilderRemoteSync` and traverses through them creating timing data for each test class from the executed tests. If there is no data at the location an empty `ultimateSuite` will be returned.

```
def ultimateSuiteBuilder() {
    echo "Syncing SuiteBuilder locally to ${firebaseLocalSuiteBuilderSync}"
    sh("mkdir ${firebaseLocalSuiteBuilderSync}")
    sh("gsutil -m rsync -r -x '.*(?<!.xml)\$' ${firebaseResultsBucket}/${firebaseSuiteBuilderRemoteSync} ${firebaseLocalSuiteBuilderSync}")
    def resultFiles = findFiles(glob: "**/${firebaseLocalSuiteBuilderSync}/**/*.xml")
    def suites = [:]
    def ultimateSuite = [:]
    def suite
    resultFiles.each { resultFile ->
        if(resultFile.getName().contains(".xml")) {
            testsuite = readFile(resultFile.path)
            suite = new XmlParser().parseText(testsuite)
            suites = [:]
            testClasses = suite.depthFirst().findAll { it.name() == 'testcase' }
            testClasses.each { testClass ->
                if (suites.containsKey(testClass['@classname'])) {
                    time = (suites[testClass['@classname']] as double)
                    suites[testClass['@classname']] = time + (testClass['@time'] as double)
                } else {
                    suites[testClass['@classname']] = (testClass['@time'] as double)
                }
            }
            suites.each { key, value ->
                if (ultimateSuite.containsKey(key) && value > (ultimateSuite[key] as double)) {
                    ultimateSuite[key] = value
                } else if (!ultimateSuite.containsKey(key)) {
                    ultimateSuite[key] = value
                }
            }
        }
    }
    return ultimateSuite
}
```

### Building test suites `testCaseBuilder()`

Now that we have the `ultimateSuite` we can divide tests in `suitePath` for the current run. `DEFAULT_SUITE_TIME_LIMIT` is how long we want each suite to take. Since we run the suites in parallel, `DEFAULT_SUITE_TIME_LIMIT` is also how long we expect the firebase tests to finish in. For new tests, or tests missing in the `ultimateSuite` we use `DEFAULT_TEST_TIME` seconds. And if the `ultimateSuite` is empty, we divide the tests into `DEFAULT_SUITE_NOS` suites. 

```
def testCaseBuilder(ultimateSuite, suitePath, avoids) {
    testClassesInTest = findFiles(glob: suitePath)
    def classNameCollections=[:]
    def testsInSuite=[:]
    def totalTime = 0
    def suiteNumber = 1
    def nodeValue = ""
    def time = 0.0
    def DEFAULT_SUITE_NOS=4
    def DEFAULT_TEST_TIME=20
    def DEFAULT_SUITE_TIME_LIMIT=600
    if (ultimateSuite.size()!=0) {
        testClassesInTest.each { testClassx ->
            if (testClassx.getName().endsWith("Test.kt") && !avoids.contains(testClassx.getName())) {
                testName = testClassx.getPath()[<strip unwanted characters to match classname in firebase results .xml file>].replace('/', '.') //format class names to use in gcloud firebase test android run
                if (ultimateSuite[testName]) {
                    time = ultimateSuite[testName] as double
                } else {
                    time = DEFAULT_TEST_TIME
                }
                totalTime = totalTime + time
                testsInSuite[testName.toString()] = time
            }
        }
        suiteCount = Math.ceil(totalTime / DEFAULT_SUITE_TIME_LIMIT)
        eachSuiteTime = totalTime / suiteCount
        time = 0.0
        testsInSuite.each { key, value ->
            if (suiteNumber < (Math.ceil(time / eachSuiteTime)).intValue()) {
                nodeValue = nodeValue + "class " + key + ","
                time = time + (value as double)
                classNameCollections["Suite" + suiteNumber] = nodeValue.toString()[0..-2]
                suiteNumber = (Math.ceil(time / eachSuiteTime)).intValue()
                nodeValue = ""
            } else {
                nodeValue = nodeValue + "class " + key + ","
                time = time + (value as double)
            }
        }
        if (nodeValue != "") {
            classNameCollections["Suite" + suiteNumber] = nodeValue.toString()[0..-2]
        }
    } else {
        counter = 1
        testClassesInTest.each { testClassx ->
            if (testClassx.getName().endsWith("Test.kt") && !avoids.contains(testClassx.getName())) {
                testName = testClassx.getPath()[testClassx.getPath().lastIndexOf("com/example")..-4].replace('/', '.')
                nodeValue = (classNameCollections["Suite" + (((counter%DEFAULT_SUITE_NOS)as int)+1)] != null) ? classNameCollections["Suite" + (((counter%DEFAULT_SUITE_NOS)as int)+1)] : ""
                classNameCollections["Suite" + (((counter%DEFAULT_SUITE_NOS)as int)+1)] =  nodeValue + "class " + testName + ","
                counter = counter + 1
            }
        }
        classNameCollections.each { key, value ->
            classNameCollections[key] = value.toString()[0..-2]
        }
    }
    return classNameCollections
}
```

### Construct firebase test run calls `transformIntoStep()`

This step is about composing the script for the parallel calls.

```
def transformIntoStep(automattionType, inputKey, inputValue) {
    return {
        echo ("Executing ${automattionType} ${inputKey}")
        resultParallel = sh returnStatus: true, script: "gcloud firebase test android run firebase_test_lab.yaml:test-devices --project <> --type instrumentation --app <> --test <> --use-orchestrator --test-targets \'${inputValue}\' --results-history-name=<> --results-bucket=${firebaseResultsBucket} --results-dir=${firebaseResultsBucketDir}${env.BUILD_NUMBER}/${automattionType}/${inputKey}"
        if (resultParallel != 0) {
            echo ("${automattionType} ${inputKey} failed.")
            currentBuild.result = 'FAILED'
        } else {
            echo ("${automattionType} ${inputKey} passed.")
        }
    }
}
```

Once the tests are finished and is a successful build, sync the results to `firebaseSuiteBuilderRemoteSync` . The reason why we do not want to sync results from failing runs are they can throw timings out.

```
echo "Sync test results to ${firebaseSuiteBuilderRemoteSync}"
sh("gsutil -m rsync -r -x '.*(?<!.xml)\$' ${firebaseResultsBucket}/${firebaseResultsBucketDir}${env.BUILD_NUMBER} ${firebaseResultsBucket}/${firebaseSuiteBuilderRemoteSync}")
```









