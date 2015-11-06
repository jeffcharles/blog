---
title: The Number Switcher 3000
---

The Number Switcher 3000 is an application for switching how my apartment's buzzer gets forwarded. There's an option to forward it to my phone, my roommate's phone, or have it automatically buzz the caller in. This blog post is a retrospective on some of the technology decisions that I made. The source code is available at <https://github.com/jeffcharles/number-switcher-3000> under a 2-clause BSD license.

## Architecture overview

When someone calls using the apartment buzzer, the phone number the buzzer contacts is one provisioned by Twilio, which is a company that provides programmatic interfaces for sending and receiving phone calls as well as text messages. The phone number's handler is set up to read XML instructions meant for Twilio present in an AWS S3 bucket. Those XML instructions are what indicate which number to forward the call to or which dial tone to echo back. The XML instructions are placed in the S3 bucket by a Node.js server running inside a Docker container on AWS Elastic Beanstalk. The server also serves up a front-end in the form of single-page JavaScript application built with React and Redux which is used for selecting which number to forward to or whether to automatically dial in callers. CircleCI is used as a continuous integration and deployment solution for pushing out server-side updates as they're available.

## Twilio

Twilio has been really great to work with so far. Their APIs are clearly documented and I haven't run into any issues with their pricing structure yet. I'd happily recommend them to anyone looking for something IVR or SMS related.

## AWS S3

S3 made logical sense as a place to get and store TwiML instructions. The Node AWS SDK made interacting with it simple. There are other object stores out there but since I decided to host the compute on AWS, S3 just seemed the easy decision.

## AWS Elastic Beanstalk

Hosting on Elastic Beanstalk made it really easy to get a continuous delivery pipeline set up. It's easy to get set up within an hour if you're using a public Docker image. One potential problem is that you're paying EC2 pricing unless you're covered by the free tier. Realistically, the only ways to have done this cheaper would be to host on Heroku's free tier, use one of Joyent's lower-end Triton containers, or host along-side other containers on the EC2 instance. I wasn't sure if I was going to have the Number Switcher dynamically respond to incoming phone calls initially so Heroku's free tier's required recharging time and cold startup lack of responsiveness made it less attractive. I wasn't aware of Joyent's public cloud offerings at the time I made the decision but I may try them out in the future to see if I can cut my costs for future projects. I'm not sure how easy it would be to get continuous delivery set up with Joyent though.
