---
layout: post
title: "Continuous Delivery for the Discerning Game Developer"
permalink: /posts/continuous-delivery-for-games/
---

Continuous Delivery is a set of software development practices for automating the process of taking a change submitted to source control and making it ready for production. This means fully automating the process of building the product, testing it, and preparing the build for release. At a high level I'd summarize the goal of CD as "Any commit to trunk can go into production". Effectively what we want to end up with is this:

<object data="../../assets/magic.svg" type="image/svg+xml"></object>

If you're not used to working on a project that has a CD pipeline setup this can sound somewhat absurd, especially if you're used to the process of preparing a build for production taking days or weeks to complete. There's also a common perception in software development that software quality and stability come at a direct cost to development velocity, so the idea that investing heavily in things like test automation, which are generally perceived as efforts to increase quality and stability, can actually speed up development is unintuitive to many people. However, there's a large body of empirical evidence demonstrating that practices like CD improve both software stability *and* development velocity, as well as having beneficial knock-on effects like reducing employee burnout.

I'm not going to be going further into the business case for CD here, but for anyone who's interested in seeing more of the details then I highly recommend checking out the Accelerate State of DevOps reports. The State of DevOps Report is an ongoing research study into how different software development practices (with an emphasis on DevOps) impact software delivery and operational performance. [The latest one is the 2019 report](https://services.google.com/fh/files/misc/state-of-devops-2019.pdf), and demonstrates a clear, causal link between practices like CD and SDO performance.

Broadly speaking, a CD pipeline will have the following steps:

<object data="../../assets/overview.svg" type="image/svg+xml"></object>

I'm going to go through each of these stages and talk about what they require to be fully implemented, and what development practices can be used achieve success in those areas. I'm also going to touch on some additional topics that aren't _strictly_ a part of CD, but are part of the broader software development process and can be part of a positive feedback loop when combined with a robust CD pipeline.

# Integrating Changes

Everything starts with a commit: Some change to the underlying source code or data that drives the product. In the CD pipeline, the first step after making the commit is to integrate that change into the mainline codebase for the project. This is a practice called "trunk based development", where all work is done directly to your project's trunk branch (or on a short-lived feature branch that's merged into trunk regularly). This keeps the effort of merging changes into trunk minimal, and avoids cases where large merge conflicts cause delays.

For the most part this practice is standard these days; Even projects that don't intentionally follow "trunk based development" as a practice use a workflow that is fairly similar. There's also generally no automation that needs to be setup in order to enable it, since modern version control tools handle the workflow for you. But there's a key piece of nuance here that I do want to highlight: Even large changes need to be implemented in a trunk-based manner, and (this is the important part) incomplete work should be regularly merged back into your trunk.

When working on relatively small or scope-constrained pieces of work there's generally no issue following this practice. But things can be more complicated when working on something that's larger in scope, such as adding a major feature or doing some kind of large-scale refactoring. Even for teams that generally follow trunk-based development, it can be tempting to do large chunks of work on a branch before merging the work into trunk. In order for a CD pipeline to work effectively, it's important to avoid this and instead follow practices that allow even major changes to be broken down and merged into the mainline piecemeal.

In short: Merging unfinished work is good, actually! We as developers often have an aversion to merging in partially-completed work, especially if what we're merging in is non-functional. However, the only real issue with merging incomplete work is if:

* It disrupts development in some way, e.g. by causing build or test failures, or by exposing unfinished work to internal testers.
* It disrupts the end user in some way, by being surfaced to users before the work was finished.

But with the right development practices, even very large pieces of work can be broken into small pieces and implemented in an iterative way without disrupting development or causing problems for users. Following this approach ensures that build and testing functionality provided by the CD pipeline can provide feedback early and often, and means we can avoid making large, disruptive changes.

There are a lot of approaches that can be taken when it comes to breaking up large changes, and the right one depends heavily on context, but I'll go over a few now as examples:

* **Avoid exposing new functionality at all** until it's ready to be integrated in the game. For example, if a new feature is intended to be accessible through a button in a menu, don't add that button until the feature has been fully implemented. Instead you can add a debug-only way of accessing the feature (e.g. an option in a debug menu or a debug-only keyboard shortcut). This allows the functionality to still be merged into the codebase before it's done without risking exposing unfinished functionality to players.
* **Hide changes behind feature flags**. If you can't fully hide the new functionality, such as in cases where you're making tweaks to an existing feature, build the system to make the new functionality toggle-able and use feature flags or configuration options to determine when to enable the feature.
* **Branch by Abstraction**. For large refactoring work you'll often be completely replacing an existing system with a completely new implementation. In this case, you need a way to continue to use the old implementation while the new one is in development. To do this, build out a layer of abstraction between the functionality that's being refactored and the code that uses it that will allow you to swap out the underlying implementation. This will allow you to still test out the new implementation while leaving the old implementation in production until you're ready to remove it. Once the new implementation is done, the old version and the abstraction layer can be removed.

# Build and Deployment

The next step depends on what type of application you're working with and what technologies its built with, but will generally some be kind of build or deployment step, possibly a combination of the two. For many projects, some build step is necessary in order to take the raw source code for the project and convert it into a format that can be run (though this may not be necessary if your application is written in an interpreted language like Python or JavaScript). Once the application has been built, there's usually some kind of deployment step that's needed in order to make the new build accessible. For server applications this will mean deploying the new build into a development environment. For client applications (i.e. ones that are run by the end user directly) there is likely some steps needed to distribute the build to the people who need it, whether that's making it available internally to the development team or uploading it to your distribution platform of choice.

As with the integration step it's pretty common these days for this step to already be automated, at least at a basic level. So what I want to focus on here are the nuances that are important for ensuring that your CD pipeline is working effectively:

* **The entire process needs to be automated** (short of actually releasing the build)!
* **Run the entire process on every commit!**

One of the driving philosophies of CD is "if something is painful, do it more often". While it's common to have a basic build process run on every commit, it's also common to only run the full release build pipeline when preparing to actually do a release. As a result, issues in the release build process don't get caught until the worst possible time: When you're trying to get a release out. Similarly, it's common to leave steps like uploading builds to release platforms as manual steps since they are performed infrequently, and the effort of automating them is seen as being more costly than continuing to do it by hand.

But if you fully automate the process and run the full process on every commit, all of the benefits of CD get applied to your release build process as well as your daily dev build process. If an issue comes up with your release builds, you find out as soon as the problem is introduced and can fix it well before it has the potential to cause a delay. This approach also allows you to do away with things like manual release branches, since every commit to trunk will produce a viable release candidate.

I expect that the biggest objection to this approach is that for many products the release pipeline is far too slow to run on every commit: Upwards of an hour, possibly taking several hours for large projects. This is a valid issue, but not an insurmountable one. In the most extreme cases running the entire release build pipeline can take so long that it would take the entire day, which would nullify the benefit of running on every commit. In these cases I recommend running release builds nightly, since that's still much better than waiting until you're actually doing a release to run the release build. For less extreme cases, say build times of an hour or so, an alternate approach is to run your builds in multiple stages. I talk about this approach more when talking about test automation.

# Testing

Testing is probably the most critical part of the entire CD pipeline. Having a robust, automated test suite is the key that allows you to be confident that the product works as intended after any given change, allowing you to work quickly with confidence. However, it's also often the hardest piece to implement effectively. Building out an effective test suite takes a substantial amount of engineering and QA effort over a long period of time, and when starting from scratch it can be hard to know where to start or to see the value that will come from that effort. I'm going to try to dig into some key pieces of the "how" and "why" of automated testing in order to make the prospect less intimidating.

## Continuous Testing

The first piece that I want to emphasize is Continuous Testing. Traditional, manual testing approaches involve having a human perform tests periodically at various points in time: Engineers will perform ad hoc tests as they implement functionality, and QA testers will perform both ad hoc tests and more rigorous tests based on pre-made test plans. However, there's a fundamental limit on how frequently and how thoroughly manual tests can be performed. As the scope of a product grows and more functionality is added, the amount of work needed to fully test every piece of functionality grows exponentially. For even relatively small applications it's simply impossible for the full test suite to be run manually with any degree of frequency. As a result, manual tests tend to be limited to regular smoke tests, with more thorough regression tests being performed only when necessary.

Automated testing makes truly continuous testing possible, since it's often possible to run the entire test suite after every change. This makes for a huge improvement over manual testing for a number of reasons:

* **Quicker feedback**, since tests are run immediately after a commit without needing to wait for a QA tester to be available. It's often easy to identify exactly which change introduced a test failure with automated testing.
* **Finer-grain testing can be run**. Manual testing can only really test the game from the perspective of a player, which means they can only catch when things break in fairly obvious ways. Automated tests can target individual pieces of code directly in a way that simply can't be done by manual testing.
* **More consistent results**. Manual testing is always subject to human error, which means test results can be inconsistent.
* **Easier to check edge cases**. One of the big advantages of automated tests is that they can cover the less common cases that are often missed by the more general smoke testing that manual testers do regularly. Uncommon cases are the most likely to break as a result of day-to-day changes, since they're often not covered by the ad hoc testing done by engineers and designers. Automated tests can consistently verify such cases after every change.
* **Improved working conditions for manual QA testers!** I'll go into this in more detail in a little bit, but one of the biggest advantages of automated testing is how it frees up manual testers. Rather than constantly having to smoke test the game in order to catch regressions, or constantly having to deal with instability and breakage, testers can focus on things like exploratory testing and user experience testing, things that only a human tester can do effectively.

In order to setup continuous testing, there's only really two conditions that need to be met:

* You must run your suite of automated tests as part of your CD pipeline after every commit.
* You must fail the pipeline and reject the build if any tests fail.

Even with a fairly small test suite, there's immense value in running those tests in this way. Whatever pieces are covered by automated tests, no matter how small, will be tested thoroughly and consistently after every change.

## Test-Driven Development

Of course, the larger and more thorough your test suite the more reliably it will catch bugs as they're introduced. However, getting to that point can be difficult, especially if you're looking to add test coverage to an existing project. To help build up test coverage, I recommend following an approach called "Test-Driven Development". The basic idea is that for any given change you want to make to the project, you write a test for the new expected behavior _before_ actually making the change. The test will fail when you first write it, but will pass once you've correctly implemented the change in question. There are a number of benefits to this approach:

* It gradually builds up test coverage. The effort of building up a test suite is spread over the entire development process, rather than trying to sit down and build an entire test suite all at once.
* It tests the tests. When writing a test to cover an existing piece of functionality, it can be hard to tell if the test is actually testing the right thing, and if the test doesn't fail when the underlying functionality is broken the test isn't providing any value. With test driven development, you write tests at a point where you know the underlying functionality isn't working, so if the test doesn't fail at that point then you know there's something wrong with the test.
* It encourages developers to account for edge cases from the beginning. The act of writing tests encourages you to think about what edge cases need to be covered and how the code should handle those cases. Putting in the time to write tests for those edge cases first means that it's easier for developers to ensure that they've fully handled all those cases when they move on to implementation.

I especially like this approach for dealing with bug fixes. When working on a new feature, writing tests ahead of time can be difficult. For a large feature, the sheer number of potential tests to write can be overwhelming and it can be hard to figure out what tests would be the most valuable to write at the start. Plus, if the feature is still being prototyped you might not even know for sure what the expected functionality is, and trying to write tests at that phase of development can be both frustrating and disruptive to the prototyping process. But with bug fixes, you can be 100% confident that every test you write adds immediate value since it's always covering a bug that we know has come up in practice. It also adds a lot of confidence to the fix, since we have a test that should pass to confirm that the fix worked.

## Architecting Code for Testability

One critical thing to keep in mind is that code needs to be written in such a way that it is amenable to testing. In order to be testable, code needs to have the following properties:

* **Deterministic** - The functionality needs to behave the same way given the same inputs and context every time.
* **Controllable** - Any inputs taken by the code must be fully controllable by the test environment, such that the exact same conditions can be used to run the test every time. If the code depends on external systems in an uncontrollable way, then it introduces ways for the tests to fail inconsistently.
* **Independent** - The code needs to be able to be run independently from other systems that it would otherwise interact with when running normally. Tests *can* be written to cover interactions between multiple systems (called "integration tests"), but even then you need to be able to limit the test to only the subsystems in question without needing to pull in other, unrelated systems.

These properties aren't hard to achieve, but it's also also easy to write code that doesn't adhere to them if you're not actively focused on making your code testable.

It's also worth noting that some kinds of functionality will be easier to setup in this way than others. When it comes to games, code related to the game's visuals and world state tend to be relatively hard to test, since the game world is an inherent piece of shared state, and is often both an input and an output for a given piece of code. Where possible, it's helpful to separate "business logic" from "view logic", such that you can test the underlying functionality separately from the logic for controlling the game's visuals. Even for more complex game logic, the underlying functionality will have all of the above properties once it's separated and can be tested on its own.

## Testing Art and Data

At Synapse we build games that are highly data-driven, and changes to the game's data can be a source of bugs as much as changes to the game's code. As such, testing the game's data and ensuring that everything is configured correctly is key to having thorough test coverage. Fortunately, testing data is relatively easy! Data can be loaded in isolation of most of the game systems, and it's generally relatively simple to write tests for specific properties that the data needs to have. In the best case, you can reuse the game's code to write tests that directly verify that the input data works as expected when used by the game. But even if your setup doesn't allow for that, writing separate tests for the data is still fairly easy to do.

Games also have a much heavier emphasis on art assets than many other applications. Fortunately, art assets can be treated fairly similarly to data in terms of automated testing: While we can't do much to test that the assets look right, there's often specific requirements for how art assets are configured and added to the project, and _those_ parts can be tested automatically. With engineers, artists, and designers all potentially making tweaks to the project at the same time, it can be especially beneficial to have tests covering art assets and game data, since more people touching a project introduces more places for bugs to be introduced.

## Verify Changes Before Merging

One key point I haven't talked about yet is _when_ to run your automated test suite.

The ideal setup is to perform all tests before changes are merged into your project's trunk. As I mentioned at the beginning, the goal of CD is that any commit to trunk can go into production. That invariant can't be maintained if tests are only run after committing to trunk. Instead, the better approach is to make changes to a branch first. Tests are performed on the branch and the change is only merged once tests have passed. For programmers this often involves making a "pull request" or "merge request", and is tied directly into the code review process such that changes require both manual approval and automated verification before being merged.

Things get trickier once you add artists and designers into the mix, since the standard pull request process used by engineers is cumbersome for changes that don't need to go through the full code review process. An alternate approach that can be used here is to have artists and designers commit to a separate branch off of trunk. The test suite can be run after each commit, and changes can be merged to trunk automatically if the tests pass.

However, it's not always going to be possible to run all tests when merging to trunk. Some tests may simply take too long to run to do so after every commit. Similarly, if your project takes a long time to build (as is often the case for games), any tests that need to be run after the build finishes will be slow to run as well. When faced with these cases, the first thing you should always do it try to speed up the tests. If the tests can be simplified in some way that allows them to still catch failures while running quickly, then doing that is your best bet. The more frequently tests are run, the more value they have, so maximizing the set of tests that can be run after every commit is your best bet for having an effective test suite.

But even still, there will always be some tests that simply take to long (or are too flaky) to run after every commit without being disruptive. For these there are two main options:

* **Run tests in multiple stages**. After a commit, first run the quick tests and allow the change to be accepted if those pass. Once the initial tests pass, kick off a second stage for the slower tests. The second stage won't necessarily run for every commit, rather it runs as fast as it can, starting again with the latest commit after the previous batch finishes. This may mean that multiple commits are bundled into a single test run, but this will still ensure that the tests are being run as quickly as they can be. This works well for tests that take several minutes to run, but are otherwise still reliable.
* **Run the tests nightly**, or otherwise on an automatic schedule. This approach should generally be your last resort, since tests that are run on a schedule, rather than in response to a change to the project, need to be checked manually and can't be used to automatically gate changes. However, running tests nightly can be useful in some cases:
  * Tests that take a *really* long time, on the order of hours.
  * Tests that can be flaky and may fail even when nothing is wrong. You should be cautious about including tests like this at all, since test failures can be easy to dismiss as random failures even if they're catching actual bugs. But if you have such tests cases that are genuinely useful, then running them nightly is probably the best approach to avoid random test failures from causing disruption to normal development.
  * Exploratory tests that are looking for new bugs. Some testing approaches, generally called "fuzz testing" or "gremlin testing", attempt to interact with the product in semi-random ways in order to discover crashes and other bugs. These tests generally need a long time to run (hours or days) and can fail unpredictably, so it's only practical to run them overnight and review any issues that they uncovered later.

This brings us to the question of how we respond to failures in the CD pipeline. If we're setup to follow the ideal case of "tests are run before merging to trunk", then generally the way to handle failure is pretty obvious: Whoever was making the change sees that their change was rejected, they fix whatever caused the tests to fail, then once tests pass again they're good to merge. For these cases the impact of the test failure is minimal since it hasn't been merged into trunk and so won't disrupt development, and it's clear who exactly needs to address the breakage.

However, for tests that are run as a second stage or after merging, we have the possibility of failure for changes that have already been merged to trunk. In these cases, it's important make fixing trunk the top priority for the team. This doesn't mean that every person on the team has to stop what they're doing until the issue is fixed, but _someone_ needs to immediately focus on fixing it, and anyone else who's help is needed should prioritize helping. This is part of the reason why running tests before merging is so important: Failures that are caught before merging are far less disruptive than those caught after merging, so catching as many issues as possible as early as possible is key to keeping the development process smooth.

## A Note on Manual QA

Earlier I talked about the advantages of automated testing and the advantages it has over manual testing for certain kinds of testing. But I really want to be clear that automated testing is NOT a substitute for manual testing. Rather, automated testing allows your manual testers to work far more effectively than they could otherwise.

On a project without automated testing, manual QA efforts are a constant uphill battle against breakage and regressions, and most of our testers' time is spent just making sure the game still works. This is a problem for a few reasons:

* It's a huge time sink, since smoke testing and regression testing needs to be done nearly constantly as changes are made to the project.
* Delays and disruptions are common, since bugs are constantly being introduced. This means that QA testers rarely have time to focus on other work like writing test plans.
* Finalizing a release is difficult and stressful since bugs are often found last minute and there's a lot of pressure on the QA team to approve a build on time, something which is often entirely out of their control.

Automated testing resolves these issues by handling the most rote, tedious forms of testing and establishing a baseline of stability. This removes a lot of the stress that comes from working on an unstable project, and frees testers up for the kinds of work that only manual testers can do:

* **User experience testing**. QA testers can give feedback not just on whether or not something works, but how it's experienced from a player's perspective. This means they can identify if things are confusing from a player's perspective, or things that otherwise don't line up with how we want player's to experience the game.
* **Exploratory testing**. Automated testing can ensure that old bugs never come back, but it's not really able to find new bugs. Manual testers know how to poke and prod at features in order to find ways to break them, and any bugs they uncover can get added to the automated regression testing suite in order to ensure that those bugs never come back.
* **Designing test cases**. Manual QA can be included early in the design and implementation phases in order to ensure that edge cases and potential bugs are caught before they ever make it into the game. These efforts then translate directly into building out automated testing for any edge cases that QA identifies.

The end result of all this is a more stable game, a better final product, and a much, much happier QA team.

# Conclusion

There's far more information about Continuous Delivery than I can cover in this article, but I hope I've provided a reasonable high-level introduction to what a CD pipeline looks like and what work goes into building one. Continuous Delivery as a practice can provide an immense amount of value for a software development team, but it's sadly under-utilized within the game development world due to some of the unique challenges that game development projects need to contend with. I hope that more game devs begin to utilize this practice to ship higher quality games more quickly than they could otherwise.
