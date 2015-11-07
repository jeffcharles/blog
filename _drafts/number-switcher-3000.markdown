---
title: The Number Switcher 3000
---

The Number Switcher 3000 is an application for switching how my apartment's buzzer gets forwarded. There's an option to forward it to my phone, my roommate's phone, or have it automatically buzz the caller in. This blog post is a retrospective on some of the technology decisions that I made. The source code is available at <https://github.com/jeffcharles/number-switcher-3000> under a 2-clause BSD license.

## Architecture overview

When someone calls using the apartment buzzer, the phone number the buzzer contacts is one provisioned by Twilio, which is a company that provides programmatic interfaces for sending and receiving phone calls as well as text messages. The phone number's handler is set up to read XML instructions meant for Twilio present in an AWS S3 bucket. Those XML instructions are what indicate which number to forward the call to or which dial tone to echo back. The XML instructions are placed in the S3 bucket by a Node.js server running inside a Docker container on AWS Elastic Beanstalk. The server also serves up a front-end in the form of single-page JavaScript application built with React and Redux which is used for selecting which number to forward to or whether to automatically dial in callers. CircleCI is used as a continuous integration and deployment solution for pushing out server-side updates as they're available.

## Continuous integration and deployment overview

The project uses branch protection on GitHub so all code changes need to go through a build on CircleCI before they can be merged. The integration process that runs on those changes and the master branch is defined in a YAML file in the repository. Essentially an `npm install` is run to install all of the project's dependencies and then an `npm run ci` is run which runs a linter, builds the client-side code, runs integration tests against the server code, and runs an end-to-end test with Chrome using Selenium. On merge into master, the same process runs, then a Docker image is built using the same dependencies pruned to ones for production and pushed to the public Docker registry, and then Elastic Beanstalk is instructed to run the newly pushed Docker image. The whole process on a merge to master takes less than ten minutes to finish.

## Twilio

Twilio has been really great to work with so far. Their APIs are clearly documented and I haven't run into any issues with their pricing structure yet. I'd happily recommend them to anyone looking for something IVR or SMS related.

## AWS S3

S3 made logical sense as a place to get and store TwiML instructions. The Node AWS SDK made interacting with it simple. There are other object stores out there but since I decided to host the compute on AWS, S3 just seemed the easy decision.

## AWS Elastic Beanstalk

Hosting on Elastic Beanstalk made it really easy to get a continuous delivery pipeline set up. It's easy to get set up within an hour if you're using a public Docker image and you're already familiar with AWS. One concern that often comes up with hosting on AWS is that you're paying EC2 pricing unless you're covered by the free tier. Realistically, the only ways to have done this cheaper would be to host on Heroku's free tier, Microsoft Azure's free tier, use Google App Engine's f1-micro, use one of Joyent's lower-end Triton containers, or host along-side other containers on the EC2 instance. I wasn't sure if I was going to have the Number Switcher dynamically respond to incoming phone calls initially so Heroku's free tier's required recharging time and cold startup lack of responsiveness made it less attractive. Microsoft Azure's free tier seems very limited and not at all intended for non-dev sites. I didn't originally think of App Engine as an option but they do have an interesting price point with their f1-micro. I wasn't aware of Joyent's public cloud offerings at the time I made the decision but I may try them out in the future to see if I can cut my costs for future projects. I'm not sure how easy it would be to get continuous delivery set up with Google or Joyent though. I'm also curious to see how Amazon will price their t2.nano offering. The pricing story on EC2 is actually really compelling if you reserve instances.

Here's the deployment script that gets run:

{% highlight bash %}
{% raw %}
#!/usr/bin/env bash

set -euo pipefail

export AWS_DEFAULT_REGION=us-east-1

VERSION=circle-$(date +%s)-${CIRCLE_SHA1}
ARCHIVE=${VERSION}.zip

docker login -e $DOCKER_EMAIL -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
docker build -t jeffreycharles/number-switcher-3000:${VERSION} .
docker push jeffreycharles/number-switcher-3000

cat << EOF > Dockerrun.aws.json
{
  "AWSEBDockerrunVersion": "1",
  "Image": { "Name": "jeffreycharles/number-switcher-3000:${VERSION}" },
  "Ports": [{ "ContainerPort": "3000" }]
}
EOF

zip $ARCHIVE Dockerrun.aws.json
aws s3 cp $ARCHIVE s3://number-switcher-3000-deployments/${ARCHIVE}
aws elasticbeanstalk create-application-version \
  --application-name number-switcher-3000 \
  --version-label $VERSION \
  --source-bundle S3Bucket='number-switcher-3000-deployments',S3Key="${ARCHIVE}"
aws elasticbeanstalk update-environment \
  --environment-name number-switcher-3000 \
  --version-label $VERSION
{% endraw %}
{% endhighlight %}

## Docker

Docker helps keep dependencies and provisioning consistent between my continuous integration server, production, and my local dev machine. It's much easier to work with compared to Ansible or Puppet. Definitely would want to ship using it again.

## Node.js

Other than serving up the landing page for the application, the backend code is entirely auth, validation, transformation, and persisting. There's pretty much no compute or local storage which makes it a very natural fit for Node.js. As a bonus, Node enabled me to make the app universal in that the web UI renders in a useful state on first load as opposed to most single page apps where the web UI needs to immediately contact the server again for data before it displays anything useful to the user. For applications like the Number Switcher, Node is a great fit and a pleasure to work with.

## React

I find React makes writing view code pretty simple compared to Angular or using plain old jQuery. Babel and Webpack make working with JSX relatively simple. Modeling the application state is a bit tricky but I've found that Redux does an effective enough job. One thing I don't like in React compared to Angular is dealing with basic form data. Angular's data bindings make it very straightforward to capture the form input in an object and then have a controller fire it off in an XHR on submit. React and Redux require wiring up each field to an action watching for change and then wiring each action up to a reducer to populate the application state appropriately. Conceptually it's not any more complicated but there is more code to write and keep track of.

## Redux

Redux is currently my favoured Flux-like data modeling solution. Unlike regular Flux, Reflux, or Flummox, it represents application state as a tree where changes are made through reducers (i.e., given a state tree and the result of an action, return the new state tree). Naturally this makes unit testing stores ridiculously easy compared to stores in traditional Flux. There's also a bit of an ecosystem around Redux with packages for things like thunks which enable straightforward handling for actions that have results that are asynchronous. Redux's React integrations are also top notch. Another advantage that makes Redux stand out is how easily it runs server-side and how that enables hydration of the client-side state tree on initial page load. This was a lot more pleasant to work with than Flummox's model which required each store to implement dehydration and rehydration functions that were pretty much identical. Getting the data to hydrate with is a bit more tricky and I'll go more into that later.

## Immutable.js

I found Immutable somewhat unpleasant to work with despite having a strong preference for immutable data structures. Most third-party JavaScript libraries expect to get native JavaScript arrays and objects so there's a lot of conversions to get things to work with them. The code you have to write when using this library also doesn't look particularly idiomatic which makes it easy to use incorrectly. The main advantage I wanted with it was to take advantage of React's pure render functionality where React simply checks if the object passes or fails a shallow equality test when deciding if it needs to re-render. It's also not great that it takes 10 kilobytes of bandwidth when minified and gzipped. In a larger app, that's not a big deal but the Number Switcher only sends down about 100 kilobytes so 10 seems like a lot for something Immutable does. Having to remember to consistently use `Object.assign` and `[].concat` is a pain but it may be worth it.

## Babel

Babel is an item I feel conflicted about. It's a transpiler that converts es6 JavaScript into es5 JavaScript. I enjoyed using it on the client-side in combination with Webpack but found using it for server-side code compilation was too slow and there were unnecessary transformations applied. There are a lot of features in es6 that make code more succinct and less error-prone and I'm happy to have a tool that enables me to take advantage of that.

On the server, I used Babel's require hook and found that it took around five or six seconds for my site to reload after a change despite having the Babel cache enabled and running. This is a long enough delay that it interrupts my flow and annoys me. I also wish there was an easy way to inform Babel that it was transpiling code for Node v4 and didn't need to make a number of the transformations that it did. Another thing to be aware of is that you cannot use rewire or rewire-global with es6 modules which means you may need to revisit how you do dependency injection for unit testing.
