---
title: "Lessons from my first side project"
date: 2024-03-07T10:56:17-03:00
tags: ["Software"]
---
The following are a few reflections on the death of my first serious [side project](https://github.com/zoftko/felf).
While it technically worked and was deployed, I never got it to a point where I could call it
production ready. This tempted me to call it a failure but I learnt a few things which will be
listed down below. It was not too bad after all.

# The Idea
The main idea was to monitor the size of binaries, specifically ELF binaries. This was aimed mainly
at embedded systems, where space is limited and if you get carried away, your project may not even
fit in Flash Memory.

I came up with the idea while working on a few embedded projects where I was curious about how
my code was affecting program size. Tracking the project's binary size across time and posting
the difference on Pull Requests seemed like a useful thing back then. That is how felf (fat elf),
my side project was born.

I wanted the project to integrate with existing code platforms like GitHub or GitLab.

# Execution
I chose Spring Boot as my main framework since I wanted to improve my Java skills, it also took
care of OAuth2 and JDBC interactions, letting me focus on the business logic. Overall the coding
experience on the backend was good. I mainly faced problems with the Frontend and GitHub App
integration.

## Frontend
I don't like designing UIs, I can appreciate a well designed one, but usually prefer to stick with
CLIs. This makes designing the front end very tedious. Once I get a general idea it is not too hard
to code it. I used [Thymeleaf](https://www.thymeleaf.org/), [daisyUI](https://daisyui.com/)
and [htmx](https://htmx.org/).

With this I was able to get an acceptable (I think) UI, however there was not even a proper
landing page and I was not really happy with it. This was an itch I was able to sweep under the rug
for a while but it eventually came back and demotivated me.

## GitHub Integration
I chose to create a GitHub app since that would let me retrieve Pull Request events and manage
comments where the diffs would get posted. Thinking back this probably over-complicated the whole
thing. I had to:

* Manage App Installations.
* Use an App token to get an Installation token.
* Use the Installation token to verify Repository information and manage comments.
* Manage Webhooks to keep data up to date.
* Authenticate users with OAuth2.

It was too much. I could probably do it all with the `GITHUB_TOKEN` that is already provided
in workflows. It would save me the whole UI (yay) since the server could be purely API based.
I would simply need to validate the GITHUB_TOKEN and store the analysis results.

# Things I learnt
Since I over-complicated the whole thing, I got to learn multiple things:

## Multi Image Docker Builds
I was able to build the app with an Eclipse JDK container and copy it
to a lean Eclipse JRE container. This made it easy to use something like Digital Ocean's App
Platform and instantly deploy new versions as I pushed code.

## Multi Module Maven Builds
I played around with making my own seeder to populate databases with
random test data. To do this I needed to reuse my Entities. This forced me to perform a multi
module maven setup, creating a module that contained all entities and could be imported as a
dependency on the server and seeder modules.

## Writing Github Actions
I wrote a custom Github Action that analyzed and pushed the results to the backend. I chose to
create a Javascript Action and was amazed by the ease of it. The SDK provided by GitHub is really
great.

# Why It died
I over-complicated the project which eventually made it tedious. I was able to get a POC and deploy
it. It worked fine and comments were automatically being published on PRs, however the UI was
unfinished and that bugged me a lot. I also stopped working on embedded systems for a while, so the
motivation to track binary sizes was lost. Had the project been simpler I probably would have
finished earlier and been happier with the project, preventing its dead.

# Conclusions
Side projects are fun. I have made lots of them but this was the first serious attempt at making
something that would be actually deployed. It is not mandatory to finish them, but if you want to
have a finished side project with acceptable quality, you really need to keep it simple and fun.

I may come back in the future to try something similar but with a much simpler workflow. For the
time being I am happy with the things I learnt. It sounds cheesy but it does seem like after all,
failure is the quickest path to success.
