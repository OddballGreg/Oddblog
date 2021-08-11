---
title: Tools Of The Trade
permalink: /tools_of_the_trade/
---

## Bread And Butter

*These Software Development tools represent the bread of butter of my day to day development work, and are what I have the most experience in using in professional and side work.*

<ins>**Ruby**</ins>

Having worked with Ruby professionally for 4 years, I've found great joy in learning the ins and outs of the language while leveraging to build many varieties of web and cli applications or generalized scripts with Ruby. Naturally it's almost impossible to mention a career of working with Ruby without due credited mention of Ruby On Rails, but I enjoy the language enough to have started painting outside the lines with it. 

Noteable **non-Rails** highlights:

- An entire development toolchain written in Ruby using Rake and liberal use of the Parrallel gem to manage the building, testing, linting and running of a large set of docker containers on one's local machine with minimal effort. This was greatly useful in the early days of a new project being co developed by junior team members who were unfamiliar with docker and Rails servers.
- Numerous examples of scripting tied to CI automations, installations and notifications.
- An automated SDLC monitoring tool that would use the API of the our version control platform to check each project for out of date docker image versions, outdated dependencies, or new security CVE's that have been found relating to those applications. It also greeted the team each morning with a quippy quote and the weather.
- A CLI based Empire Simulation using Ruby fake value generation gems and a little inspiration from Paradox Grand Strategy games.

<ins>**Ruby On Rails**</ins>

My first experience with Ruby was writing a Ruby On Rails application for my very first internship to assist the sales staff. As such it goes without saying that as a web framework I've enjoyed learning it's ins and outs over the course of the past 4 years, watching it progress through various interesting iterations over the course of building at least 15 distinct Ruby On Rails based applications/services, with a further 5 legacy applications which I did not assist in the construction of from inception, but assisted in the development, enhancement and maintenance of. These applications have spanned the course of Rails versions from 3.0 to 6.1, making use of all (and sorely missing) the full gamut of Rails feature set.

Noteable Highlights:

- Online Store product scraping and cross searching platform to seek out the best deals across all online offerings
- Near Realtime APN tracking and management platform and API for enterprise cellular data.
- A* Pathinding Algorithm Demonstration platform, which would generate and solve an A* pathfind problem in the space of a 3 second http request-response cycle.

Naturally, there are some applications that might not necessarily be mentioned.

<ins>**Docker & Docker Compose**</ins>

Docker is an application containerization protocol that has made such a marked impact on the way that I develop software, that in the occasions where I've helped teach some of my more junior colleagues, it's one of the first things I showed them how to make use of to effectively simplify their environmental setup efforts.

Gone are the days of reconfiguring Postgres hba_conf files to trust unix port connections, or questioning why your Redis server no longer wants to start, when both these services can be started with a command as simple as `docker run` with but a few arguments. It's also an immense asset for decomplexifying development of applications across a team, especially if they might be developing with different local enviroments.

Finally as a deployment asset, it's usefulness is maultifauceted, especially when coupled with Docker Compose to easily configure entire compositions of services to spool up, modify or update at a moments notice. (This is also rather useful for easy development as well.)

As far back as early 2018, Docker has been my defacto platform for all my development and deployments, so much so that a recent reinstall of my Ubuntu distribution had me momentarily scratching my head as to where my Ruby runtime was... only to realize I'd grown so used to working in containers, I'd not needed to install it at any point yet.

<ins>**Rspec**</ins>

One of the two defacto Ruby testing frameworks, I've used Rpsec for effective testing of countless Ruby and Rails applications at unit, system and integration test levels in an elegant and pleasantly understandable syntax. As a highly versatile tool in the ensurance of code stability, I've used many additions to Rspec to further empower it's utility and ability to provide development value, such as:

- FactoryBot for effective data instantiation on a per test basis (Fixtures are a nightmare)
- Capybara for integrated monolith application system tests
- Faker for easy random information generation
- SimpleCov for effective code (and more recently, branch) test coverage

<ins>**Sidekiq & Redis**</ins>

The dynamic duo, Sidekiq as an effective asynchronous job management and processing system atop a Redis cache database is a time tested mechanism for building highly dynamic and performant systems that can perform complicated operations without impacting User Experience. Reliable, consistent and well supported, there's very few Rails applications I've built that hasn't included these two.

A particular high point included these use of Sidekiq and Redis to facilitate realtime data packet queueing and ingestion for a bespoke APN management and tracking system. This system was able to maintain tens of thousands of update packets per hour without so much as a shudder.

<ins>**Postgres**</ins>

Postgres, the seemingly standard RDBMS choice of Rails users as far as I've so far experienced. It's a relational database that defaults to utf8mb_4 and doesn't trust unauthenticated port connections by default, and if those 2 things have never been a problem for you, you've probably not worked on a Mysql backed Rails app.

Also, did you know you can query Postgres with Regex? And I don't mean pull all the records and checks a field for regex matches, I mean pass a regex query string to Postgres and let it return to you all matches. All in a shockingly performant manner too! I built a stimulus interface which would present all items that matched a user written regex in real time, allowing you to modify the regex until the desired results were found.

<ins>**Watir & Selenium**</ins>

When it comes to web scraping for information or integration testing JS SPA's hooked up to ruby API's, Watir and Selenium are invaluable tools for automating browser interactions. Coupled with Rspec these can formulate the basis of effective integration testing in a familiar testing format for most Ruby devs.

WIP, more to come...

<!-- <ins>**Ansible**</ins>

<ins>**Version Control**</ins>
<sup>*Github/Gitlab/Bitbucket*</sup>

<ins>**CI/CD**</ins>
<sup>*CircleCi/GitlabRunners/Jenkins/GithubActions*</sup>

<ins>**CloudAfrica**</ins>

<ins>**Websockets (ActionCable)**</ins>

<ins>**Terraform**</ins>

<ins>**AWS**</ins>

<ins>**Neptune Graph DB**</ins>

<hr>

## Interested, Learning or Partial Experience

*These are tools I'm less experienced in, am actively learning to use more effectively, or would be interested in actively adopting.*

<ins>**WASM**</ins>

<ins>**Service Workers**</ins>

<ins>**Crystal**</ins>

<ins>**Lucky**</ins>

<ins>**Sinatra**</ins>

<ins>**Cloudformation**</ins>

<ins>**Mysql**</ins>

<ins>**MongoDB**</ins>

<hr>

## Past Exposure, Mild Usage, or Unpreffered

*These are tools I've used to some degree in the past to some small amount but might not have have actively continued using either for lack of reason, or due to actively preferring not to.*

<ins>**Javascript**</ins>

<ins>**Nodejs**</ins>

<ins>**C**</ins>

<ins>**Heroku**</ins>

<ins>**Python**</ins> -->






