---
layout: post
title:  "PrestaShop in 2019 and beyond, part 2: The Pain Points"
subtitle: "aka What needs to be improved"
date:   2019-09-11 09:30:00
authors: [ PabloBorowicz ]
icon: icon-compass
tags: [1.7, architecture]
---

This is the second in a [series of articles][introduction] we introduced earlier this year, that aims to describe where we are, where we are going, and some ideas on how we'll get there.

# The Pain points
_(or "What needs to be improved")_

In the [previous part][previous-article], we described what the current architecture looks like. This article aims to analyze what are the main "pain points", that is, fundamental problems that are dragging the project behind.

But before getting to that, let's address the elephant in the room. After reading the previous part in this series, probably the number one thing you thought of was "why _the hell_ is this so complicated?". 

There are many reasons for that, but I think there's one that can be found at the root of it. This will be a long article, so please bear with me.

## Introduction

First of all, I believe there is an all-too-common misunderstanding about very nature of PrestaShop. Many people think of PrestaShop as a product: something that you download and install to build your shop on your browser. But it's not. Or at least I don't see it that way, specially now that [Ready is a thing](https://www.prestashop.com/en/prestashop-ready).

When I meet people and tell them what I do for a living, I usually describe PrestaShop as an _open-source platform for developing e-commerce websites_. This is in my view, a more accurate description of what this project is about.

Yes, I have met people who download it and use it as-is, but I think they belong to a declining minority. To my best understanding, most people who are choosing to use the PrestaShop project in 2019 (and who succeed in doing so) do it to kickstart their own development. They are usually agencies or experts, and often they are also part of our rich community of module and theme developers who sell in the [Addons Marketplace](https://addons.prestashop.com).

This means that that we, the people who work with PrestaShop's Core code, we have to keep tree kinds of "users" into account:

1. The shoppers that buy things on PrestaShop shops,
2. ... the merchants that run them (and ultimately, pay for),
3. ... and the **developers** that work using PrestaShop to build sites, or develop modules and themes.

This third "user", is very, _very_ important to us. [I mean it like Steve Ballmer](https://www.youtube.com/watch?v=1VgVJpVx9bc).

It's not just because we _love_ developers (incidentally, we do!). It's because PrestaShop cannot survive without them. As such, we want to encourage them to develop for our platform. 

As platform developers, we face a big challenge that most developers (or at least, web developers) usually don't need to care about: **backwards compatibility**.

Developers who work with PrestaShop (or any platform for that matter) expect things to work in a certain, predictable way. Either because the vendor explicitly described them that way through documentation, or because they spent time figuring out how it works on their own, in many cases through excruciating trial-and-error and reverse engineering of undocumented code. The bottom line is, once they get something working, developers expect it to _continue working that way_—and it's a natural thing to do.

This expectation about how two independent pieces of software are supposed to work together is usually based on an _interface_, a kind of contract or agreement between the two parties, that can be either explicit (ie. clearly defined) or not. If it's implicit, then it's an _understanding_ of sorts, where one of the parties basically assumes the terms while the the other party does neither validate nor challenge them. 

Of course, after having invested time and money in making something work, developers won't appreciate having to update their code regularly every time there's a new platform version—ideally, it should "just work". Therefore, it goes without saying that introducing any change that breaks this agreed-upon _interface_ (a "breaking change") won't help with keeping developers interested in adopting a new version of said platform. Again, this is true for _all_ platforms.

As a result, many platform vendors—PrestaShop included—will go out of their way to avoid introducing breaking changes as much as possible. But this is a double-edged sword.

It is well-known that, in time, [software rots](https://en.wikipedia.org/wiki/Software_rot). Most developers agree that the best medicine for avoiding software decay is applying frequent [refactoring](https://refactoring.com/), because once code rots for good, it becomes unmaintainable and has to be rewritten—and history shows us that [rewriting from scratch is a terrible idea](https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/).

However, refactoring can only get you so far until it starts to require introducing breaking changes. And once you hit that brick wall, you're stuck: either you let that part of the code rot or you introduce breaking changes that will undermine the adoption of your platform. Or worse, you introduce an _new_ subsystem that does the same thing as the old one, and keep the previous one for compatibility, consequently increasing the overall software complexity.

That's one of the top reasons PrestaShop's architecture has become so complex. The constant need to provide as much backwards compatibility as possible, even when doing so was detrimental to the platform's overall technical quality, has substantially undermined the project's capacity to move forward.

That said, let's dive in...

## Plan

* [An aging architecture](#an-aging-architecture)
* [No clearly-defined API](#no-clearly-defined-api)
* [1.7 was released too soon](#17-was-released-too-soon)
	* No clear scope, no clear vision (where do we go?)
	* Customization forgotten (no overrides)
	* Duplicate systems (Sf vs legacy, twig vs smarty, translations, bo theme, vue)
* Too many dependencies with external services
* Testing is hard
* Obsolete dependencies (PHP 7+, jquery)
* No specification
* Poor documentation

## An aging architecture

PrestaShop has deeply-rooted design issues that can be explained simply by the age of its architecture.

Web technologies have evolved massively in the last years, and industry standards are now very different compared to what they were 10 years ago, when the current foundations of PrestaShop were laid out. Back then, web applications were still being built using the classic backend-generated, form-centric approach that became commonplace during the [CGI-based era][cgi], which forged the modern Internet.

Fast forward to 2019, the web is a very different place. It's a mobile-first world, where experiences provided by [Progressive Web Apps][pwa] have become so rich that the frontier between a web site and a native phone app is now at its thinnest. A world where interacting with a form while offline, infinite scroll and instant interaction is now normal, and reloading a page is archaic. A world made of rich front-end applications, and simple API-based backends.

Let's face it, PrestaShop's architecture, as we know it, shows its age. PrestaShop was designed for [a world of Smarty and jQuery][legacy-controllers], but we currently live in a world of React (or Vue, or Angular) and GraphQL.

### Systemic problems

PrestaShop was also designed using design patterns that were commonplace in the PHP world 10 years ago but have now become obsolete.

#### Bottom up, data-centric design

PrestaShop's architecture was designed in a database-centric, [bottom-up way](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design#Software_development). This is a classic design pattern where you start building small blocks, then build bigger blocks based on those. Typically it starts by designing the database schema, then creating your models based on that, then the code that manipulates those models.

If done right, this kind of design approach can be good for reusability when working in an iterative, exploratory way. But it can be a nightmare for platform maintainers that need to keep interoperability and code longevity in mind. Let me explain.

In this approach, components are designed and built _before_ they are actually used, meaning their behavior is mainly based on _guesses_ regarding what future needs they will address. Therefore, it's usually not long until an unforeseen use case is found for which the component's design is not well-adapted.

When working in an iterative fashion this is not a problem, because you can go back to the drawing board, redesign the component and deploy it to production. It's part of the workflow. But when you're building a development platform, where this will most likely happen _after_ it is first released, this means that the maintainer will have to find a way to alter the component so that it addresses this particular behavior, **without introducing breaking changes**. This is how spaghetti code creeps in and it's also one of the most common ways technical debt is created.

In PrestaShop, this problem is exacerbated by widespread data micro management, where state changes are handled through direct model manipulation (eg. "get instance of Product, change property value, save", or worse, an SQL update query) instead of transitions. In theory, models should know little about each other, and controllers should be slim on logic. Who's responsible for ensuring that products inside an order exist and are in stock, for instance? If someone update a product, who's in charge of updating search indexes? Currently there's no unique structure in the core that guarantees this vital system-wide coherence.

#### Global state

PrestaShop's legacy code is mainly made out of static classes which are heavily dependent on global or semi-global state, like `Context` and `Db`. While this approach could initially have seemed like a good idea (no need to think about dependencies, "cleaner", easy-to-use code that "just works"), it actually brings [a great deal of classic, well-known problems](https://softwareengineering.stackexchange.com/a/148154), including two that are very bad for any project:

* Global state and tightly interweaved dependencies are bug-prone, meaning that bugs are harder to fix and regressions are more likely to creep in.
* It's much harder to create automated tests, which aggravates the risk of undetected regressions.

The new architecture introduced in PrestaShop 1.7 ultimately aims at removing all static classes and replace them by components designed following the SOLID principles. However, due to the enormous amount of work it represents and the number of breaking changes it will introduce, most of the lower-level components are and will still be present for the whole lifetime of 1.7. This means that unfortunately, `Context`, `Db`, `ObjectModel` and the such aren't going away anytime soon—nor are the problems they bring.

### Technical debt

Technical debt is a problem for all mature projects, and PrestaShop is not the exception there. As stated before, the best way to avoid software rot is to apply frequent refactoring. But if refactoring is limited by breaking changes, then it cannot be applied properly and technical debt starts piling up.

Even though continuous efforts are being made to avoid this, having to circle around breaking changes ultimately increases complexity. And indeed, the current architecture has become quite complex, with a legacy part that contains a great number of crisscrossing dependencies, and a new-generation part that still requires following many not-so-obvious rules and many workarounds. In addition, more layers means more code to load in memory, which in some cases can also result in a performance penalty.


## No clearly-defined API

As we said before, interoperability is a big subject in platform development. Since refactoring is, in the long term, unavoidable, breaking changes are bound to occur sooner or later.

Remember back in 2015, when [PrestaShop started following SemVer][semver-article]? The Semantic Versioning (SemVer) [specification](https://semver.org/) defines that only major versions can introduce backwards incompatible API changes, while minor and patch versions must ensure backwards compatibility. 

This is good for interoperability: it means that as long as integrations respect the software's API, they can be sure they will keep working with the main software, even if that software is upgraded to the next minor or patch version.

In the case of PrestaShop, merchants want to be able to upgrade their shop seamlessly and benefit from the new features and bug fixes, without having to change their theme, modules and customizations, or fear something will break down. As stated before, SemVer is supposed to ensure this, as long as third parties respect PrestaShop's API. This is true—in theory.

**But... what _exactly_ constitutes PrestaShop's API?**

Right now there's no article documenting it out anywhere. If there was, we would be able to say: "This is PrestaShop's API. It is something that we are committing on. Respect it, and enjoy forward compatibility!".

So, if it's not clearly defined anywhere... then what is this unwritten contract made up from? Let's find out.

First, we must first identify **who** are our technical stakeholders. Then, we have to establish **what** are the technical resources they need to interact with, and finally define **how** those interactions will take place. Sounds familiar? We have already depicted that in the [previous article][previous-article] (see [figure][comprehensive-overview]).

By looking at the architecture overview, we can easily pinpoint PrestaShop's four main technical stakeholders:

- Themes
- Modules
- Web service clients
- Customizations (custom code for custom functionality, that is, overrides or rewritten core code)

If we look at the lines going in and out of them, we can see the relationships between these stakeholders and the rest of the system:

- **Themes** depend on...
	- Smarty 
	- Any parent theme (like Classic, for instance)
	- Module templates (if they "skin" those modules)
	- Data structures provided to templates
	- Core javascript library (jQuery)
- **Modules**	 depend on...
	- Smarty and/or Twig
	- Base module features provided by the Core
	- Hooks
	- Overridable templates in the BO
	- BO themes, their CSS and Javascript
	- Core classes
- **Web service clients** depend on...
	- Webservice API (which itself is tightly coupled to ObjectModel)
- **Customizations** depend on...
	- The Override system
	- Core services
	- Core classes

We have now established **who** are the stakeholders and **what** resources they need. But what about the **how**?

Well, if the **how** is not clearly defined, then we are left with two choices: either there's _no_ API, or **_everything_ used by stakeholders is the API**.

[Let that sink in for a minute](https://media.giphy.com/media/aZ3LDBs1ExsE8/giphy.gif). That's a _pretty big_ API, isn't it?

Big indeed. As we can see, the level of dependencies is _massive_. That means that essentially, PrestaShop's "API" is made out of every template file, every public member of every PHP class, and every javascript object. Oh, and assets, too.

![Use all the things](/assets/images/2019/09/use-all-the-things.jpg)
{: .center}

Of course, this level of access is fantastic for customizations. _Everything_ is right there, available for use and modification. But at what cost?

Having everything open up for use and modification severely limits PrestaShop's leeway to implement changes in the Core (be it features, bug fixes or refactoring) without introducing backwards incompatible changes.

How severely does this affect us? Here are some examples of things that _shouldn't_ be done until the next major version of PrestaShop:

- Change any public method signature in any class (rename, change its return type or structure, remove a parameter or change its type).
- Change any public property in any class.
- Rename, move, delete any class or class namespace.
- Add new requirements (like dropping support for old versions of PHP or browsers, requiring new server-side libraries...)
- Replace any subsystem (like updating libraries to new major versions or replacing a library with another).

Obviously, these requirements are incompatible with the stakes of 1.7, so we have been forced to take [some liberties][sf-upgrade] in order to be able to move forward, even if it's a rule that we try not to break if we can help it. Still, this is a recurrent pain: 

- **Want to implement a new feature**? _Better think of a way to do that in a retrocompatible way_.
- **Want to replace ObjectModels with Doctrine?** _Nope, BC break_.
- **Want to refactor that class to make it testable?** _No can do, that would require changing its signature_.
- **Want to upgrade that dependency to benefit from cool new features?** _Not if it's a major version upgrade_.
- :'(

### Everything is public

As if making every class part of our API was not enough of a problem, a lot of legacy core classes suffer from _public_-itis. It means that classes have lots of public methods and properties that needn't be.

This is particularly troublesome in PrestaShop, because once a class member has become public—even if it was a mistake—it becomes part of the public API as soon as it's released... and therefore cannot be refactored until the next major version!

This _even worse_ because of the [overrides system][overrides-system], which allows developers to replace any legacy core class with a custom one (violating the [Open–Closed principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle), by the way). Since the overrides system is based on inheritance, and inheritance grants overrides access to protected class members of the extended class, and any accessible code is considered part of the API... then **protected members of legacy core classes can be considered part of the public API as well**, because they are accessible. 

Having no clearly-defined contracts means we can't change anything without introducing a breaking change, which severely limits the options for improving PrestaShop.

## 1.7 was released too soon

PrestaShop 1.7.0 was a very ambitious release. It featured a [new architecture](/news/new-architecture-1-6-1-0/), a [new framework (Symfony)](/news/prestashop-1-7-and-symfony/), a new Back Office theme (later based on our [UI kit](/news/PrestaShop-UI-Kit/)), a [new default Front Office theme](/news/new-theme-1-7-introduction/) and [Starter Theme](/starter-theme-kickoff/), a [new Product page experience](/news/product-page-evolution/), a [new Modules page experience](/module-page-awakens/), a [new translation system](/news/new-translation-system-prestashop-17/) and [use of CLDR](/cldr-composer-jshint/), among many other features. 

That's an impressive list. Unfortunately, that objective was proven to be too ambitious, and the result was an underwhelming initial release.

Before continuing, it's important to say that following this release and between 2016 and 2018, most of the original team that developed PrestaShop 1.7 left the company or was reassigned. In 2017, a new CTO (Aurélien Pelletier) and I arrived at PrestaShop. We agreed on a new long-term vision that would point the project in the right direction, and started building a new team to make it happen. Early 2018, with the support of the newly appointed PrestaShop CEO, Alexandre Eruimy, the team started growing and the project started picking up speed again.

Now, even though PrestaShop 1.7 is [getting better and better with every release](https://www.webbax.ch/2019/01/10/unboxing-decouverte-prestashop-1-7-5-0/), we find that many decisions that were taken during early development of 1.7 were ultimately proven to be bad calls, since they are making it harder to move the project forward.

### No clear vision

In my view, PrestaShop 1.7.0 was developed and released without a clear, global vision of what the ultimate objective of this major version was, nor how long it would take to do it. In addition, a tradition of oral, word of mouth knowledge sharing at PrestaShop meant that if there ever was a vision, it was unknown to most and is now forever lost. This is something we are meaning to change, and this series is part of that.

### Overeagerness to introduce new things

Too many technical projects were started in parallel before being thoroughly thought through and were released before being properly finished. As I said before, this would be acceptable in a SaaS software where you can work iteratively and release often, but doesn't work in a development platform like PrestaShop where upgrades are costly and scarce. Of course, releasing this way resulted in a giant, arguably well-deserved backlash from the community, who described PrestaShop 1.7 as "unfinished software".

Here's an example. Back in 2017, we announced that we had started [introducing Vue JS in the Back Office][vuejs], supported by a BO API, which would be used in the Stocks and Translations pages.

While I think this is the way of the future (no spoilers!), in hindsight, I believe that it was too early to introduce it in 1.7, and will probably have to be redone.

Don't start screaming at me just yet! Let me explain why.

First of all, if you take a glance at the [architecture overview][comprehensive-overview], you'll notice that there are two different stacks in the Back Office: the _Legacy stack_ and the _New stack_. The former is pretty much unchanged since 1.6, while the latter is essentially made out of Symfony controllers and Symfony forms, rendered with twig. Two stacks, one that will progressively replace the other. Right? 

**Wrong.** If you look closer, you'll see that the _New stack_ actually has not one, but **two** implementations: the Symfony one that I described, and another one, based on VueJS and an API.

So, we don't have _two_ different ways of doing things, we have _three_. 

This is justified because the initial idea was that VueJS was supposed to be the way the migration to Symfony should have continued starting on 1.7.3.0. But we chose to freeze that and continue working with Symfony forms and Twig. Why?

Because _customization_.

PrestaShop owes its success mainly to its ability to be extended and customized. For this, developers rely on extension points that allow them to add behavior in specific points, as well as conventions that allow them to replace certain bits of the software by their own.

We think that this aspect wasn't thoroughly accounted for when designing the VueJS stack, as current extension methods do not work with VueJS pages. And of course they don't, because they behave in a drastically different way. So **there's no defined way to customize a VueJS page in PrestaShop right now**.

If we want to allow these pages to be customizable, we have to rethink how, and provide a stable, clearly-defined way of doing it. That is a very long process that cannot be improvised in the middle of a major migration, so we have to go back to the drawing board and start over. In the meantime, we need to make those pages extensible again.

Consequently, our only option right now to make all three systems converge is to continue down the Symfony path before switching to something else. This decision will make more sense once you read the next two articles in this series.

In the meantime, we will continue to add VueJS in the Back Office, but only as standalone components like the [live search engine results preview component][serp] that we added in the Product Page in 1.7.5.0.


### Duplicate systems

Many of the new systems in PrestaShop 1.7 don't replace the previous ones, but feature a new, semi-independent way of doing the same thing that coexists with the previous way, kept for backwards compatibility. 

Having more than one way of doing a given thing is bad for many reasons. First, it introduces cognitive friction, because as a developer you have to think about in when you should use one or the other. Second, because a business or dependency change in one must be replicated in the other. And third, if they don't produce the same output or side-effects for some reason, then it might produce systemwide inconsistencies.

There are several examples of this in PrestaShop 1.7. One of them is the "Core–Adapter–Legacy" architecture. Let's explore four problems tied to this example:

#### 1. Shared responsibility

As described in the previous article, in order to allow old code to progressively be replaced by new code, the Core namespace conventions prohibits developers from using legacy classes directly in Core classes. When developers need legacy functionality in a Core class, it needs to be wrapped in an Adapter class that can be then injected into the Core class.

**The problem:** Since all the Adapter does is wrap legacy features in a non-static class, we now have two classes that do the same: one in Adapter, one in legacy.

#### 2. Different implementations/side effects

The idea behind Adapters is that they should eventually be replaced by Core code that reimplements the feature, allowing us to delete the legacy class later. However, some developers chose to partially reimplement some legacy features in Adapters, but introducing a _slightly_ different behavior.

**The problem:** Not only do we have two classes that do the same thing, but they do it in a _different_ way and produce _different_ side effects. Which one should we use, and when?

#### 3. Little or no added value

Looking at the code, we can see that many hastily made Adapters have been designed [as dumb bridges to legacy code][bad-example], even replicating method signatures and design flaws of their legacy counterparts. Some even have static methods!

**The problem:** Many adapters do not even add any value to legacy classes and exist only to comply with the "no use of legacy classes in new code".

#### 4. Hard dependencies

To make matters worse, most Adapters do not even implement an interface, which makes it impossible to replace them with Core reimplementations without modifying the dependent classes.

**The problem:** Most adapters are hard dependencies and cannot be replaced without changing the dependent code. 

#### Bonus: No single source of truth

And finally, there's the matter of Database access. Since most interactions with the database are performed through ObjectModels, and since ObjectModels are too difficult to reimplement with an Adapter (we want to get rid of them anyway), some developers reimplemented them using Doctrine.

**The problem:** Even models have been duplicated. Some of them _aren't even in the adapter namespace_ and work in a _completely different way_... so there's no single source of truth anymore.

There are many other duplicate systems in 1.7:

- Symfony controllers vs Legacy controllers
- Twig vs Smarty
- Symfony translation system vs legacy
- New BO theme (based on bootstrap 4) for Symfony pages vs legacy one (based on bootstrap 3) for legacy pages


* Customization forgotten (no overrides)
* Duplicate systems (Sf vs legacy, twig vs smarty, translations, bo theme, vue)

### Legacy layout



### Translation system

Customization forgotten (no overrides)

# Too many dependencies with external services

# Obsolete dependencies (PHP 7+, jquery)

# Testing is hard










---















## Complexity and rigidity

The current architecture has become very complex, with a legacy part that contains a great number of crisscrossing dependencies, and a new-generation part that follows many not-so-obvious rules and still requires a lot of workarounds. In addition, more layers means more code to load in memory, which in some cases can also result in a performance penalty.

On top of that, since legacy classes are still present, many of the fundamental problems that existed in the previous versions persist in 1.7:

1. Static classes, global state (including static cache), hidden dependencies (inline instantiation, or worse... static dependencies).
2. Procedural code, thousand-line classes with too many responsibilities.
3. Everything's public.

I won't bore you right now describing all the reasons why the first two issues are problematic. There are already lots of books and articles about that out there (if you're interested, you can start [here](https://en.wikipedia.org/wiki/SOLID)). I'll get back to the third in a minute.

These issues are typical of a certain way of developing that was commonplace in the PHP world for a long time, and I have seen them in lots of projects the age of PrestaShop. They can fly under the radar and be safely ignored for quite a while... as long as you don't intend to write automated tests.

### Quality assurance is hard; Quality assurance in PrestaShop is _very hard_

![Testing is doubting][testing-is-doubting]{: style="height:400px"}
_Testing is doubting, by [CommitStrip](http://www.commitstrip.com/)_
{: .text-center }

This is one of the classic _consequences_ of complex, static, globally-stateful code. Since components are so tightly interweaved and depend on global state, putting the system in a state that allows to test a feature in an isolated and predictable way for automated testing is really, really hard. In some cases, dependencies and hidden state can become extremely difficult and time-consuming to understand, specially when dealing with static in-memory cache (ie. variables that maintain their value between calls to a same method on a same thread).

Unsurprisingly, hard-to-test code also leads to complex, hard-to-maintain test scripts.

Let's see an example. Until a some months ago, here's what we had:

- A mix of unit/integration/survival tests, based on PhpUnit.
- A suite of End-to-end (E2E) tests, based on Mocha and Selenium.

Ideally, we should be able to classify our tests into three types:

- **Unit** – Where a single, completely isolated class is tested at a time.
- **Integration** – Where a subsystem is tested in its entirety, with carefully selected dependencies replaced by test doubles.
- **E2E** – Where the system is tested in its entirety from a user standpoint, using browser automation.

We have been struggling with automated tests failing unexpectedly, not because the feature they tested had stopped working, but because of diverse, unrelated technical issues:

- Some unit tests aren't really unit (usually because they tested several classes at once).
- Many unit tests that aren't unit were also extremely hard to understand and maintain, because of all the boilerplate code needed to put the shop in the state we needed for testing.
- Some tests were producing side-effects that made other tests fail because of unexpected/inconsistent state (usually due to hidden cache in legacy code).
- E2E tests failed randomly because of timeouts (Selenium + WebDriverIO issues).

We believe that bugs are a natural side-effect of changing things, and that automated tests are the only way to be able to consistently move forward without breaking stuff... but what happens when the tests themselves are breaking up constantly for unknown reasons?

Based on this evidence, we recently decided to:

- Move all PhpUnit tests into a `legacy-tests` directory while we clean them up.
- [Introduce Behat][behat] for Integration, human-readable tests.
- Progressively refactor tests into either Unit (completely isolated) or Integration tests.
- Move E2E test from Selenium + Webdriver to [Puppeteer][puppeteer].

This will help us get a better grasp on tests. But it will remain hard to automate them as long as the system they test remains so complex.


## Duplicate systems

As described in the previous article, to allow old code to progressively be replaced by new code, the Core namespace conventions prohibits developers from using legacy classes directly in Core classes. When developers need legacy functionality in a Core class, it needs to be wrapped in an Adapter class that can be then injected into the Core class.

**First problem:** since all the Adapter does is to wrap legacy features in a non-static class, we now have two classes that effectively do the same.

The idea behind Adapters is that they will eventually be replaced by Core code that reimplements the feature, allowing us to delete the legacy class. Some developers chose to partially reimplement some legacy features in Adapters, but introducing a _slightly_ different implementation, while keeping the previous behavior in the legacy class for backwards compatibility.

**Second problem:** not only do we have two classes that do the same thing, but they do it in a _different_ way and produce _different_ side effects. Which one should we use, and when?

Looking at the code, we can see that many hastily-made Adapters have been designed [as dumb bridges to legacy code][bad-example], even replicating method signatures and design flaws of their legacy counterparts. They even have static methods!

**Third problem:** many adapters do not even add any value to legacy classes and exist only to comply with the "no use of legacy classes in new code".

To make matters worse, most Adapters do not even implement an interface, which makes it impossible to replace them with Core reimplementations without modifying the dependent classes.

**Fourth problem:** most adapters are hard dependencies. 

And finally, there's the matter of Database access. Since most interactions with the database are performed through ObjectModels, and since ObjectModels are too difficult to reimplement with an Adapter (and we want to get rid of them anyway), some developers reimplemented them using Doctrine.

**Fifth problem:** some models are duplicated as well, they _aren't even in the adapter namespace_ and work in a _completely different way_... so there's no single source of truth anymore.

## Bad decisions

Many decisions taken during early development of 1.7 now make it harder to work with. But some of them are not that old.

Back in 2017, we announced that we had started [introducing Vue JS in the Back Office][vuejs], supported by a BO API, which would be used in the Stocks and Translations pages.

While I think this is the way of the future (no spoilers!), in hindsight, I think it was too early to introduce it in 1.7, and will have to be rolled back.

Don't start screaming at me just yet! Let me explain why.

First, if you take a glance at the [architecture overview][comprehensive-overview], you'll notice that there are two different stacks in the Back Office: the _Legacy stack_ and the _New stack_. The former is pretty much unchanged since 1.6, while the latter is essentially made out of Symfony controllers and Symfony forms, rendered with twig. Two stacks, one that will progressively replace the other. Right? 

**Wrong.** If you look closer, you'll see that the _New stack_ actually has not one, but **two** implementations: the Symfony one that I described, and another one, based on VueJS and an API.

So, we don't have _two_ different ways of doing things, we have _three_. 

This is justified because the initial idea was that VueJS was supposed to be the way the migration to Symfony should have continued starting on 1.7.3.0. But we chose to freeze that and continue working with Symfony forms and Twig. Why?

Because _customization_.

PrestaShop owes its success mainly to its ability to be extendable and customizable. For this, developers rely on extension points that allow them to add behavior in specific points, as well as conventions that allow them to replace certain bits of the software by their own.

We think that this aspect wasn't thoroughly accounted for when designing the VueJS stack, as current extension methods do not work with VueJS pages. And of course they don't, because it behaves in a drastically different way. Let me repeat: **there's no defined way to customize a VueJS page in PrestaShop right now**.

If we want to allow this pages to be customizable, we have to rethink how these pages would be customizable, and provide a stable way of doing it. That is a very long process that cannot be improvised in the middle of a major migration, so we have to go back to the drawing board and start over. In the meantime, we need to make those pages extensible again.

Our only option right now to make all three systems converge is to continue down the Symfony path before switching to something else. This decision will make more sense once you read the next two articles in this series.

In the meantime, we will continue to add VueJS in the Back Office, but as standalone components, like the [live search engine results preview component][serp] that we added in the Product Page in 1.7.5.0.



translation
bad communication -> no clear vision
No overrides outside legacy (vs services)
synchronous backend requests to outside services


## Obsolete dependencies

Classic in BS 4 alpha 5
jquery
php 5

## Obsolete module management system

Composer vs zip
overrides


[introduction]: /news/prestashop-in-2019-and-beyond-introduction/
[previous-article]: /news/prestashop-in-2019-and-beyond-part-1-current-architecture/
[testing-is-doubting]: /assets/images/2019/03/tester-cest-douter.jpg
[behat]: https://github.com/PrestaShop/PrestaShop/pull/12090
[puppeteer]: https://developers.google.com/web/tools/puppeteer/
[semver-article]: /news/a-more-semantic-versioning-scheme/
[comprehensive-overview]: /assets/images/2019/02/architecture-comprehensive-overview-current.jpg
[sf-upgrade]: /news/prestashop-1-7-is-moving-to-symfony-3-4-and-php-5-6/
[overrides-system]: https://devdocs.prestashop.com/1.7/modules/concepts/overrides/
[cgi]: https://en.wikipedia.org/wiki/Common_Gateway_Interface
[pwa]: https://developers.google.com/web/progressive-web-apps/
[legacy-controllers]: https://devdocs.prestashop.com/1.7/development/architecture/legacy/legacy-controllers/
[bad-example]: https://github.com/PrestaShop/PrestaShop/blob/95683248751795b1e927445d57abaf45708fea09/src/Adapter/Attribute/AttributeDataProvider.php
[vuejs]: /news/introducing-vuejs-symfony-api/
[serp]: /prestashop-1-7-5-0-available/#product-page
