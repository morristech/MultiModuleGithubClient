[![Build Status](https://travis-ci.com/frogermcs/MultiModuleGithubClient.svg?branch=master)](https://travis-ci.com/frogermcs/MultiModuleGithubClient)

# MultiModuleGithubClient

Breaking the monolith to microservices is a well-known concept to make backend solutions extendable and maintainable in a scale, by bigger teams. Since mobile apps have become more complex, very often developed by teams of tens of software engineers this concept also grows in mobile platforms. There are many benefits from having apps split into modules/features/libraries:

* features can be developed independently
* project structure is cleaner
* building process can be way faster (e.g., running unit tests on a module can be a matter of seconds, instead of minutes for the entire project)
* great starting point for [instant apps](https://developer.android.com/topic/google-play-instant/)

This is the example project which resolves (or at least workarounds) the most common problems with the multi-module Android app.

## Project structure

The project is a straightforward Github API client containing 3 screens (user search, repositories list, repository details). For the sake of simplicity each screen is a separate module:

* app - application module containing main app screen (user search)
* repositories - repositories list
* repository - repository detail
* base - module containing a code shared between all modules

![Project structue](https://raw.githubusercontent.com/frogermcs/MultiModuleGithubClient/master/docs/img/app_diagram.png "Project structure")


## Dagger 2

Recently there is no recommended Dagger 2 configuration for multi-module Android project. Some software engineers recommend exposing Dagger modules from Android feature module and use them in Components maintained only in the main App module. This project implements the second way - each feature module has its Component and dependencies tree. All of them depends on Base Component (created in Base module). All have their scope and subcomponents.
This approach was proposed by my colleague [@dbarwacz](https://github.com/dbarwacz) and recently is heavily used in [Azimo](https://azimo.com) Android application.

Here are some highlights from it:

* `BaseComponent` is used as a dependency in feature components (e.g. `RepositoriesFeatureComponent`, `AppComponent`...). It means that all dependencies that are used in  needs to be publicly exposed in `BaseComponent` interface.
* Local components, like `SplashActivityComponent` are subcomponents of feature component (`SplashActivityComponents` is a subcomponent of `AppComponent`).
* Each module has its own Scope (e.g. `RepositoryFeatureScope`, `AppScope`). Effectively they define singletons - they live as long as components, which are maintained by classes: `AppComponentWrapper`, `RepositoryFeatureComponentWrapper` which are singletons... . To have better control on Scopes lifecycle, a good idea would be to add `release()` method to ComponentWrappers.

For the rest take a look at the code - should be self-explaining. As this is just self-invented setup (that works on production!), all kind of feedback is warmly welcomed.

![Dagger structue](https://raw.githubusercontent.com/frogermcs/MultiModuleGithubClient/master/docs/img/dagger_diagram.png "Dagger structure")

## Unit Testing

Project contains some example unit tests for presenter classes.

### Gradle

To run all unit tests from all modules at once execute:

```
./gradlew testDebugUnitTest
```

In console you can see:

```
...
> Task :app:testDebugUnitTest
com.frogermcs.multimodulegithubclient.SplashActivityPresenterTest > testNavigation_whenUserLoaded_thenShouldNavigateToRepositoriesList PASSED
com.frogermcs.multimodulegithubclient.SplashActivityPresenterTest > testErrorHandling_whenErrorOccuredWhileLoadingUser_thenShouldShowValidationError PASSED
com.frogermcs.multimodulegithubclient.SplashActivityPresenterTest > testValidation_whenUserNameIsInvalid_thenShouldShowValidationError PASSED
com.frogermcs.multimodulegithubclient.SplashActivityPresenterTest > testValidation_whenUserNameValid_thenShouldLoadUser PASSED
com.frogermcs.multimodulegithubclient.SplashActivityPresenterTest > testInit_shouldLogLaunchedScreenIntoAnalytics PASSED

> Task :features:base:testDebugUnitTest
com.frogermcs.multimodulegithubclient.base.ExampleUnitTest > addition_isCorrect PASSED

> Task :features:repositories:testDebugUnitTest
com.frogermcs.multimodulegithubclient.repositories.RepositoriesListActivityPresenterTest > testRepositories_whenRepositoriesAreLoaded_thenShouldBePresented PASSED
com.frogermcs.multimodulegithubclient.repositories.RepositoriesListActivityPresenterTest > testInit_shouldLoadRepositoriesForGivenUser PASSED
com.frogermcs.multimodulegithubclient.repositories.RepositoriesListActivityPresenterTest > testNavigation_whenRepositoryClicked_thenShouldLaunchRepositoryDetails PASSED
com.frogermcs.multimodulegithubclient.repositories.RepositoriesListActivityPresenterTest > testInit_shouldLogLaunchedScreenIntoAnalytics PASSED

> Task :features:repository:testDebugUnitTest
com.frogermcs.multimodulegithubclient.repository.RepositoryDetailsActivityPresenterTest > testUnit_shouldSetRepositoryDetails PASSED
com.frogermcs.multimodulegithubclient.repository.RepositoryDetailsActivityPresenterTest > testInit_shouldSetUserName PASSED
com.frogermcs.multimodulegithubclient.repository.RepositoryDetailsActivityPresenterTest > testInit_shouldLogLaunchedScreenIntoAnalytics PASSED

BUILD SUCCESSFUL in 55s
...
```
It can be useful especially when you run tests on your CI environment. To control logs take a look at file `buildsystem/android_commons.gradle` (it is included in all feature modules).

```
testOptions {
    unitTests {
        all {
            testLogging {
                events 'passed', 'skipped', 'failed'
            }
        }
    }
}
```

In case you want to run unit test in one module, execute:

```
./gradlew clean app:testDebugUnitTest
```
or
```
/gradlew clean feature:repository:testDebugUnitTest
```

### Android Studio

The repository contains shared configurations for running Unit Tests directly from Android Studio.

![Android Studio configs](https://raw.githubusercontent.com/frogermcs/MultiModuleGithubClient/master/docs/img/as_run_configurations.png "Android Studio configs")

#### All tests at once

Recently it is not always possible to run all tests at once (see troubleshooting below).

#### All tests, module after module sequentially

Run `App and modules tests`. This configuration will run unit tests from all modules in separate tabs, one after another. To modify list of tests, click on `Edit Configurations` -> `App and modules tests` -> `Before Launch`.

![All tests sequential](https://raw.githubusercontent.com/frogermcs/MultiModuleGithubClient/master/docs/img/all_tests_sequential.png "All tests sequential")

#### Troubleshooting

* **Class not found: ... Empty test suite.**
There is a bug in Android Studio which prevents from launching all unit tests at once, before their code is generated (what happens after the first run of unit tests for every single module independently). For more take a look at Android Studio bug tracker: https://issuetracker.google.com/issues/111154138.

![Android Studio issues](https://raw.githubusercontent.com/frogermcs/MultiModuleGithubClient/master/docs/img/failing_all_tests.png "Android Studio issues")

## Test coverage for unit tests

The project contains additional configuration for Jacoco that enables coverage report for Unit Tests (initially Jacoco reports cover Android Instrumentation Tests).

To run tests coverage, execute:

```
./gradlew testDebugUnitTestCoverage
```

Coverage report can be found in `app/build/reports/jacoco/testDebugUnitTestCoverage/html/index.html` (there is also an .xml file in case you would like to integrate coverage report with CI/CD environment.

### Implementation details

Setting up a coverage report for Android Project isn't still straightforward and can take a couple of hours/days of exploration. Example setup in this project could be a little bit easier and more elegant, but some solutions are coded explicitly for better clarity.
Here are some highlights:

* Each module should use Jacoco plugin `apply plugin: 'jacoco'` and config (defined in `android_commons.gradle`):
```
buildTypes {
    debug {
        testCoverageEnabled true
    }
}
```

* App module defines custom Jacoco task called `testDebugUnitTestCoverage`. Entire configuration can be found in `buildsystem/jacoco.gradle`. The code should be self-explaining.

* Task `testDebugUnitTestCoverage` depends on `testDebugUnitTest` tasks (each module separately). Thanks to it all sources required for coverage report are available before gradle starts generating it (in `<module_dir>/build/...`.

![Coverage report](https://raw.githubusercontent.com/frogermcs/MultiModuleGithubClient/master/docs/img/coverage_report.png "Coverage report")
