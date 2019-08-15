+++
title = "Using load-tests as quality gate in Jenkins pipelines"
date = 2019-08-14T10:41:28+02:00
publishDate = 2019-08-20
tags = ["devops", "taurus", "jmeter", "performance", "loadtesting", "testing", "jenkins", "ci/cd", "pipelines" ]
+++

Over years [JMeter][jmeter] has always been a valuable tool in my box when dealing with performance testing. But when we've integrated those tests with our CI/CD-pipeline we always had hard times to control the pipeline outcome based on the performance results. <br/>
In other words, let the test results act as a quality gate for either promoting the build-artfiact along the pipeline or not.
<!--more-->

#### Prevent jenkins lock-in

When working with [JMeter][jmeter] there are a bunch [of good tutorials][codecentric] out there which show you how to integrate [jmeter test-suites with maven][jmeter-maven] and subsequently leaverage that integration also [within jenkins][codecentric]. I really liked that approach since it just uses the developers tools also in jenkins - easy to grasp and to try for yourself.<br/>
There are [other solutions][jmeter-jenkins-1] as well but since most of them rely on [jenkins plugins][jmeter-cloudbees] they'd introduce an undesired jenkins dependency (bearing the risk of a de facto jenkins lock-in).

__Rule of thumb__: _Always (re-)use the developers tooling within CI/CD as well._

#### Measuring is not enough

One downside of the [jmeter-maven approach][jmeter-maven] although was that there is no easy solution to break the build based on predefine thresholds (like it is possible with the [jenkins performance plugin][jmeter-cloudbees]).<br/> 
Although it provides valuable information, just charting the results (using either another [maven plugin][jmeter-graphs] or a jenkins-plugin) is simply not enough since you would have to actively check regularly for regression - a tedious task which would soon be neglected. A better solution would be if jenkins could recognize a performance degression on its own and just act accordingly (by breaking the build).

__Rule of thumb__: _Apply "[Don't call us - we call you][hollywood]" also within jenkins by breaking the build._

#### Wrapping Taurus around JMeter

While reading about [JMeter][jmeter] I finally stumbled upon [Taurus][taurus] which is, according to the webpage, an automation-friendly framework for continuous testing.<br/> 
The reason why I started to have a closer look at Taurus was its ability to _easily_ define [pass/fail criteria][taurus-passfail] for tests - like in this example:

{{< highlight yaml>}}
reporting:
- module: passfail
  criteria:
  - avg-rt of IndexPage>150ms for 10s, stop as failed
  - fail of CheckoutPage>50% for 10s, stop as failed
{{< /highlight >}}

So in case the average response time of the _IndexPage_ test is ≥10 seconds for more than 150 milliseconds the test will be aborted.

##### Loadtesting Setup

Its CLI-based nature makes Taurus a perfect fit for Jenkins pipeline integration.<br/> 
To kill two birds with one stone we created a Docker(-Agent) Image to containerize Taurus (a starting-point for such an [image is already provided by blazemeter][taurus-docker]). That image can then be used [from within Jenkins][jenkins-docker] as well as locally on each dev machine. A corresponding pipeline-snippet might look like this:

{{< highlight groovy>}}
withDockerContainer(image: 'jenkins/taurus') {
    String propPrefix = "modules.jmeter.properties"
    withCredentials([usernamePassword(credentialsId: 'loadtest-user', 
                                      passwordVariable: 'apipass', 
                                      usernameVariable: 'apiuser')]) {
        sh  "bzt -o ${propPrefix}.api_user=${apiuser}" +
            "    -o ${propPrefix}.api_password=${apipass}" +
            "    -o ${propPrefix}.api_users=10" +
            "    -o ${propPrefix}.api_rampup=30" +
            "    -o ${propPrefix}.api_loops=500" +
            " path/to/taurus.yml"
    }
}
{{< /highlight >}}

As you can see, Taurus allows us to pass through prop-values to the underlying, parametrized JMeter script - this can be used to externalize credentials but also to easily tweak the actual test-setup (i.e. adjust the number of concurrent users etc.).

When executing Taurus you'll have to reference the respective test-configuration - in our example `path/to/taurus.yml`. Since we already had a JMeter script available we refrained using Taurus' YAML-syntax to define our suite but just referenced the already existing JMX-script.

{{< highlight yaml>}}
# lists the scenarios to be executed
execution:
  - scenario: mainapi

# for props that mustn't/don't change 
modules:
  jmeter:
    properties:
      api_port: '8084'
      api_schema: 'https'

# defines each scenario - the actual test-suite is defined in 
# 'api_access.jmx' (an ordinary JMeter script)
# The names referenced in the criteria-section (i.e. Read All 
# Customer) are samples defined in api_access.jmx
scenarios:
  mainapi:
    script: api_access.jmx
    criteria:
      - avg-rt of Read All Customer>300ms for 10s, stop as failed
      - avg-rt of Read specific Customer Profile>150ms for 10s, stop as failed
      
reporting:
  - module: passfail        # let the test fail in case we don't meed the criteria
  - module: junit-xml       # will write a junit-report (which can be consumed by Jenkins)
    filename: ./junit-performance.xml
    data-source: pass-fail
  - module: final-stats     # will write a performance-report (which can be consumed by Jenkins)
    dump-xml: ./jenkins-performance.xml
{{< /highlight >}}

In the reporting section we also defined the creation of a bunch of reports that can be consumed by Jenkins as well.

With that setup we got everything we wanted:

- no lock-in - the whole tool-chain can be used by the pipeline like it is used locally (through parametrization you can change whatever aspect you'd like).
- fast feedback - as soon as a performance degression gets introduced, Taurus will break the build and Jenkins will let you know
- performance tests as quality gate - if a build passes the performance test stage the artifact is eligible to be used (from a performance point of view). If the build fails the artifact is rejected. 


[jmeter]:https://jmeter.apache.org/
[jmeter-maven]:http://jmeter.lazerycode.com/
[gatling]:https://gatling.io/
[selenium]:https://www.seleniumhq.org/
[taurus]:http://gettaurus.org/
[taurus-instal]:http://gettaurus.org/install/Installation/
[taurus-docker]:https://hub.docker.com/r/blazemeter/taurus
[taurus-executor]:http://gettaurus.org/docs/ExecutionSettings/#Executor-Types
[taurus-passfail]:http://gettaurus.org/docs/PassFail/
[codecentric]:https://blog.codecentric.de/en/2014/01/automating-jmeter-tests-maven-jenkins/
[jmeter-jenkins-1]:https://wiki.jenkins.io/display/JENKINS/How+to+run+JMeter+with+Jenkins
[jmeter-cloudbees]:https://www.cloudbees.com/blog/how-integrate-jmeter-jenkins
[jmeter-graphs]:https://github.com/codecentric/jmeter-graph-maven-plugin
[hollywood]:https://en.wikipedia.org/w/index.php?title=Hollywood_principle
[jenkins-docker]:https://jenkins.io/doc/pipeline/steps/docker-workflow/#withdockercontainer-run-build-steps-inside-a-docker-container
