# PiranhaJava

## Installation

### Overview

Piranha requires that you build your code with [Error Prone](http://errorprone.info), version 2.3.2 or higher.  See the [Error Prone documentation](http://errorprone.info/docs/installation) for instructions on getting started with Error Prone and integration with your build system.  

### Gradle

To integrate Piranha into your Java project you'll need a version of the following additions to your `build.gradle` file:

```
plugins {
  id "com.github.sherter.google-java-format" version "0.7.1"
  id "net.ltgt.errorprone" version "0.6" apply false
  id "java"
}

sourceCompatibility = "1.8"
targetCompatibility = "1.8"

dependencies {
  annotationProcessor "com.uber.piranha:piranha:0.0.3"
  errorprone "com.google.errorprone:error_prone_core:2.3.2"
  errorproneJavac "com.google.errorprone:javac:9+181-r4173-1"
}

import net.ltgt.gradle.errorprone.CheckSeverity

tasks.withType(JavaCompile) {
  options.errorprone {
      check("Piranha", CheckSeverity.WARN)
  }
  options.errorprone.errorproneArgs << "-XepPatchChecks:Piranha"
  options.errorprone.errorproneArgs << "-XepPatchLocation:IN_PLACE"
  // The lines below should be replaced by code that loads the specific flag to patch
  // and final treatment condition.
  options.errorprone.errorproneArgs << "-XepOpt:Piranha:FlagName=SAMPLE_STALE_FLAG"
  options.errorprone.errorproneArgs << "-XepOpt:Piranha:IsTreated=true"
  options.errorprone.errorproneArgs << "-XepOpt:Piranha:Config=config/piranha.properties"
}
```

The `plugins` section pulls in the [Gradle Error Prone plugin](https://github.com/tbroyer/gradle-errorprone-plugin) for Error Prone integration. In `dependencies`, the `annotationProcessor` line loads Piranha, the `errorprone` line ensures that a compatible version of Error Prone is used, and the `errorproneJavac` line is needed for JDK 8 compatibility.  

In the `tasks.withType(JavaCompile)` section, we pass some configuration options to Piranha.  First `check("Piranha", CheckSeverity.WARN)` sets Piranha issues to the warning level. Then, `option.errorprone.errorproneArgs` is used to add a set of arguments to Piranha. `XepPatchChecks:Piranha` and `-XepPatchLocation:IN_PLACE` arguments are used together to enable in-place refactoring of the code. `-XepOpt:Piranha:FlagName` is used to specify a stale flag name that is used in the code, `-XepOpt:Piranha:IsTreated` is used to specify whether the treatment (`true`) branch or the control (`false`) branch needs to be taken during refactoring. Then `-XepOpt:Piranha:Config` is used to provide the properties file which specifies the APIs and annotations that are considered for refactoring. 

The properties file has the following template: 

```
treatedMethods=treated,flagEnabled
controlMethods=flagDisabled
emptyMethods=enableFlag,disableFlag
treatmentGroupMethods=isToggleInGroup
annotations=FlagTesting
linkURL=<provide_your_url>
```

The `treatedMethods` are the APIs which correspond to the treatment behavior of the flag, `controlMethods` correspond to the control behavior of the flag. In the above example, the API `flagEnabled` corresponds to treatment behavior. Hence, when `IsTreated` flag is set to `true`, `flagEnabled(SAMPLE_STALE_FLAG)` will be evaluated to `true`. Similarly, `flagDisabled(SAMPLE_STALE_FLAG)` which corresponds to the control behavior will evaluate to `false`. 

The `emptyMethods` specify the APIs which need to be discarded from the code. For example, a statement `enableFlag(SAMPLE_STALE_FLAG);` will be deleted from the code. 

The `annotations` specify the annotations used (e.g., in unit testing) to determine treatment or control behavior. For example:

```
@FlagTesting(treated = TestExperimentName.SAMPLE_STALE_FLAG)
public void some_unit_test() { ... }
```

will be refactored to 

```
public void some_unit_test() { ... }
```

when `IsTreated` is `true`, and will be deleted completely when `IsTreated` is `false`. 

Finally, the setting `linkURL` in the properties file is to provide a URL describing the Piranha tooling and any custom configurations associated with the codebase. 


## Example refactoring

Consider a simple example

```
public class MyClass {
  private XPTest expt;
  ...
  public void foo() {
    if(expt.flagEnabled(TestExperimentName.SAMPLE_STALE_FLAG)) {
        System.out.println("Hello World");
    }
  }

  public void bar() {
    if(expt.flagDisabled(TestExperimentName.SAMPLE_STALE_FLAG)) {
        System.out.println("Hi World");
    }
  }
}
```

and the following arguments to Piranha

```
options.errorprone.errorproneArgs << "-XepOpt:Piranha:FlagName=SAMPLE_STALE_FLAG"
options.errorprone.errorproneArgs << "-XepOpt:Piranha:IsTreated=true"
options.errorprone.errorproneArgs << "-XepOpt:Piranha:Config=config/piranha.properties
```
where `piranha.properties` contains the following, 

```
treatedMethods=treated,flagEnabled
controlMethods=flagDisabled
emptyMethods=enableFlag,disableFlag
annotations=FlagTesting
linkURL=<provide_your_url>
```

the refactored output will be 

```
public class MyClass {
  private XPTest expt;
  ...
  public void foo() {
     System.out.println("Hello World");
  }
  
  public void bar() {
  }
}
```

When `IsTreated` is `false`, then the refactored output will be 

```
public class MyClass {
  private XPTest expt;
  ...
  public void foo() {
  }
  
  public void bar() {
    System.out.println("Hi World");
  }
}
```

This example is present in the [sample](https://github.com/uber/piranha/tree/master/java/sample/) directory. 
