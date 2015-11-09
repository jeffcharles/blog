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

S3 made logical sense as a place to get and store Twilio XML instructions. The Node AWS SDK made interacting with it simple. There are other object stores out there but since I decided to host the compute on AWS, using S3 was an obvious decision.

## AWS Elastic Beanstalk

Hosting on Elastic Beanstalk made it really easy to get a continuous delivery pipeline set up. It's easy to get set up within an hour if you're using a public Docker image and you're already familiar with AWS.

One concern that often comes up with hosting on AWS is that you're paying EC2 pricing unless you're covered by the free tier. Realistically, the only ways to have done this cheaper would be to host on Heroku's free tier, Microsoft Azure's free tier, use Google App Engine's f1-micro, use one of Joyent's lower-end Triton containers, use Lambda, or host along-side other containers on the EC2 instance. I wasn't sure if I was going to have the Number Switcher dynamically respond to incoming phone calls initially so Heroku's free tier's required recharging time and cold startup lack of responsiveness made it less attractive. Microsoft Azure's free tier seems very limited and not at all intended for non-dev sites. I didn't originally think of App Engine as an option but they do have an interesting price point with their f1-micro. I wasn't aware of Joyent's public cloud offerings at the time I made the decision but I may try them out in the future to see if I can cut my costs for future projects. I'm not sure how easy it would be to get continuous delivery set up with Google or Joyent though. Using Lambda was not an option since I'd made the app universal which Lambda wouldn't handle well. I'm curious to see what the upcoming t2.nano's pricing will be and whether it makes sense to reserve instances to bring AWS's prices down.

Anyway, here's the deployment script that gets run:

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

One thing that Elastic Beanstalk does not address well is blue-green deployment or at least having some sort of smoke test run after deployments or environment changes. It's possible to write your own tooling to deal with this but I wish there were more out-of-the-box support for that.

## Docker

Docker helps keep dependencies and provisioning consistent between my continuous integration server, production, and my local dev machine. It's much easier to work with compared to Ansible or Puppet. Definitely would want to ship using it again.

One area where I have room for improvement is figuring out how to effectively test the image. Options I've considered are running the integration tests twice, once against the code and then again against the container, and just running the tests against the container. The problem with running them twice is it makes the CI process take longer and running them once against the container means I'm running different tests locally compared to the CI server.

## Node.js

Other than serving up the landing page for the application, the backend code is entirely auth, validation, transformation, and persisting. There's pretty much no compute or local storage which makes it a very natural fit for Node.js. As a bonus, Node enabled me to make the app universal in that the web UI renders in a useful state on first load as opposed to most single page apps where the web UI needs to immediately contact the server again for data before it displays anything useful to the user. For applications like the Number Switcher, Node is a great fit and a pleasure to work with.

## React

I find React makes writing view code pretty simple compared to Angular or using plain old jQuery. Babel and Webpack make working with JSX relatively simple. Modeling the application state is a bit tricky but I've found that Redux does an effective enough job. One thing I don't like in React compared to Angular is dealing with basic form data. Angular's data bindings make it very straightforward to capture the form input in an object and then have a controller fire it off in an XHR on submit. React and Redux require wiring up each field to an action creator watching for change and then wiring each action up to a reducer to populate the application state appropriately. Conceptually it's not any more complicated but there is more code to write and keep track of.

## Redux

Redux is currently my favoured Flux-like data modeling solution. Unlike regular Flux, Reflux, or Flummox, it represents application state as a tree where changes are made through reducers (i.e., given a state tree and the result of an action, return the new state tree). Naturally this makes unit testing reducers easier than stores in traditional Flux. There's also a bit of an ecosystem around Redux with packages for things like thunks which enable straightforward handling for actions that have results that are asynchronous. Redux's React integrations are also top notch. Another advantage that makes Redux stand out is how easily it runs server-side and how that enables hydration of the client-side state tree on initial page load. This was a lot more pleasant to work with than Flummox's model which required each store to implement hydration functions that were pretty much identical. Getting the data to hydrate with is a bit more tricky and I'll go more into that later.

## Immutable.js

I found Immutable somewhat unpleasant to work with despite having a strong preference for immutable data structures. Most third-party JavaScript libraries expect to get native JavaScript arrays and objects so there's a lot of conversions to get things to work with them. The code you have to write when using this library also doesn't look particularly idiomatic which makes it easy to use incorrectly. The main advantage I wanted with it was to take advantage of React's pure render functionality where React simply checks if the object passes or fails a shallow equality test when deciding if it needs to re-render. It's also not great that it takes 10 kilobytes of bandwidth when minified and gzipped. In a larger app, that's not a big deal but the Number Switcher only sends down about 100 kilobytes so 10 seems like a lot for something Immutable does. Having to remember to consistently use `Object.assign` and `[].concat` is a pain but it may be worth it.

## Babel

I'm mostly positive about Babel. It's a transpiler that converts es6 JavaScript and JSX into es5 JavaScript. I enjoyed using it on the client-side in combination with Webpack but found using it for server-side code compilation was too slow and there were unnecessary transformations applied. There are a lot of features in es6 that make code more succinct and less error-prone and I'm happy to have a tool that enables me to take advantage of that.

On the server, I used Babel's require hook and found that it took around five or six seconds for my site to reload after a change despite having the Babel cache enabled and running. This is a long enough delay that it interrupts my flow and annoys me. I also wish there was an easy way to inform Babel that it was transpiling code for Node v4 and didn't need to make a number of the transformations that it did. Another thing to be aware of is that you cannot use rewire with es6 modules which means you may need to revisit how you do dependency injection for unit testing.

## Webpack

Webpack is a tool like like Browserify that converts JavaScript written in Node-style common.js to JavaScript optimized for the browser. What it does differently from Browserify is it will also handle non-JavaScript assets like LESS files pulled in through require statements in your JavaScript. The practical advantage of this approach is you get the same dead code elimination in your non-JavaScript assets as you get in your JavaScript ones (i.e., if nothing references a certain LESS file, it's not compiled and added to your site's assets). You need to require assets on a component-by-component basis in the JavaScript for that component to take advantage of this instead of having one entry point for your each type of non-JavaScript asset (i.e., don't compile your LESS with one entry-point).

Another interesting experimental feature in Webpack is hot module replacement which enables loading new JavaScript without requiring a browser refresh. I haven't had a chance to try that yet but that could make the development experience smoother.

## Supertest

`supertest` is an NPM module for testing HTTP APIs. The neat thing about it is that you don't need to have your web server running, it will take a reference to a Node http server object (like an Express serer) and handle starting and stopping it on an unused port on your behalf. You can also give it a URL to make requests to if you prefer which I could see being very useful for testing a Docker container or smoke testing a live server. `supertest-as-promised` wraps `supertest` nicely allowing you to use a promise API when chaining multiple HTTP call tests.

Here's a neat example:

{% highlight js %}
{% raw %}
it('should list login when not logged in', () =>
  request(app).get('/api')
    .set('accept', 'application/json')
    .expect(200)
    .expect('content-type', /json/)
    .expect({ actions: ['login'] })
);
{% endraw %}
{% endhighlight %}

## Webdriver.io

Webdriver is a JavaScript wrapper around Selenium. This enables browser-based end-to-end testing. Since these types of tests are difficult to maintain and tend to be prone to false negatives, I just have one test case for the Number Switcher 3000:

{% highlight js %}
{% raw %}
it('should allow login and changing the number', () =>
  browser
    .url('/')
    .waitForExist('input#loginToken')
    .then(() => browser.setValue('input#loginToken', conf.login_token))
    .then(() => browser.submitForm('input#loginToken'))
    .then(() => browser.waitForExist('input[type="radio"]:not(:checked)', 2500))
    .then(() => browser.click('input[type="radio"]:not(:checked)'))
    .then(() => browser.click('button[type="submit"]'))
    .then(() => browser.waitForExist('.savedSuccessfully', 2500))
    .then(() => browser.log('browser'))
    .then(browserLogs => {
      const nonDebugLogs =
        browserLogs.value.filter(log => log.level !== 'DEBUG');
      if (nonDebugLogs.length) {
        console.log('Browser logs:');
      }
      nonDebugLogs.forEach(log => console.log(log));
      assert.equal(nonDebugLogs.length, 0, 'Should have no log entries');
    })
);
{% endraw %}
{% endhighlight %}

Unfortunately, the documentation available through the Webdriver site was not very good. While all of the methods and options were documented, there was minimal context around given how to order calls and pretty much no complete examples.

## CircleCI

CircleCI is the third-party continuous integration server I chose to go with. The main thing that swayed me to use them over TravisCI was their support for using SSH to debug builds. I also felt that TravisCI was taking an odd direction with their approach to Docker support. Specifically that Docker builds were only supported on their legacy infrastructure and not on their new container infrastructure.

## Actions instead of hypermedia

One thing that a lot of useful APIs provide is an affordance for indicating what future state transitions are possible. As an example, it's pointless to pull down the list of phone numbers if the user isn't logged in since the server would just respond with an error status code. One way APIs go about this is through the use of hypermedia where a property is included in the object in the response body which includes a list of link relations and their corresponding hrefs and any other relevant properties. In practice, this adds effort to writing APIs and most do not include this affordance as a result.

What I decided to do with the Number Switcher was to replace hypermedia with the concept of actions on the base URI where an action essentially is like a link relation but without a link. This enables the API client to see what state transitions are possible but requires out-of-band knowledge on how to trigger those transitions. This also reduces the effort involved to provide a feed-forward affordance for the Number Switcher API since the possible state transitions are driven entirely by whether or not the user is logged in so the logic only needs to be written and tested for in location instead of on each endpoint.

The trade-off compared to full hypermedia is that the client needs out-of-band knowledge on what the URIs involved with each state transition are. I think this is reasonable because most hypermedia transitions also require out-of-band knowledge about which HTTP method is the appropriate one to use and what valid request and response bodies look like. HTML has affordances for this but there are currently no widely adopted JSON ones so there will always be a dependency on out-of-band knowledge. The presence of actions enables at least some client-side knowledge of state and acceptable state transitions which is better than the case of having neither actions nor hypermedia present.

## Complications with a universal JavaScript app

This was the first time I attempted to write a universal JavaScript app. By universal, I mean that the web UI code executes on the server and generates useful HTML and a useful state tree on the first page load as opposed to the typical single page app paradigm of the server just serving the web UI JavaScript and an HTML shell requiring the web UI code to make several HTTP requests back to the server before it's in a usable state. There are complications that come out of this approach.

The first is that you are constrained to using client-side tooling that supports server-side execution and rendering. Luckily React and Redux both work well for this approach other than the delay on development reload times when a Babel re-transpile kicks off.

The second is that you need a strategy for populating component state and all of the alternatives I evaluated had trade-offs. One approach is to have the components continue to use data loading actions that result in HTTP requests to the server on mounting. Besides resulting in more HTTP traffic for the server to handle, this approach also doesn't work well with cookie-based authentication. Requests made from the server would need to include the cookie but running the same code on the browser would result in an XHR error since that field is restricted for XHRs. The approach I chose to go with was extracting more state into the Redux state tree, extracting more of the logic out of the request handler functions and into helper functions on the server, and having the initial page load function use the helper functions directly to populate the Redux state tree and then use React to render the HTML from that as well as send down the serialized state tree. That approach works well for the Number Switcher but I suspect it wouldn't scale for larger apps because of how easy it is for the structure and types in state tree generated on the server to diverge from the state tree expected by the client side code. I think using some sort of schema or type system to define the state would be necessary. Extracting more of the state into the Redux state tree can also result in more code that's harder to follow compared to being able to keep the state in the component without involving Redux.

The third is that CDNs and pure backend-as-a-service approaches for your landing page are no longer possible since landing pages require a server-side environment capable of rendering HTML to do the rendering server-side.

One subtle benefit of having a universal app is that you can test whether a good chunk of client-side code will execute without type errors by loading pages that perform server-side rendering in different states using supertest. Requesting the page in interesting states and asserting a 200 status code is returned is quick to write and helps rule out some very obvious errors.
