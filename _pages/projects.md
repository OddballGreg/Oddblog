---
title: Projects
permalink: /projects/
---

Please note that this list of projects may be incomplete for reasons relating to sensitivity / nda or others.

## Saicom.Connect

Saicom Connect was the name for Saicom's internally developed APN and systems, built to monitor and manage a continuous stream of user data usage information. This enabled Saicom to offer bespoke services related the mobile data usage of their clients and their employees; as well as provide a valuable management capability to the sim cards used as automated failover capability for Saicom's existing VOIP solutions.

Saicom Connect was rapidly prototyped and deployed over the course of 6 months by Cameron Norman and myself using Ruby On Rails, Docker and Vue.js. It's various components were automatically deployed using a combination of CI/CD tools and Docker Compose.

## Saicom Call Saver

The Saicom Call Saver project was an algorithmic platform designed to assist Saicom's sales staff to secure VOIP line deals with prospective clients by analysing the itemised billing of their call history in order to provide a report detailing the precise amount the prospective client would have paid had their telephony services been provided by Saicom.

The call saver platform would then allow the salesperson to adjust the rates offered to the prospective client if needed in order to provide a better deal if needed.

The application could then provide additional information such as contention ratios, line concurrency, as well as aggregated stats such the full percentage the client would save by changing to Saicom. All of which could then provided to the client in a PDF report.

The Saicom Call Saver was prototyped and deployed over the course of 2 months, with additional development casually continued over the next 2 years. It was built in Ruby on Rails and deployed bare metal using Capistrano.

## Tariffic Proposal Pusher

The Tariffic Proposal Pusher was a project given to me during my first WeThinkCode_ internship by the Tariffic team as an opportunity to learn Web Development from scratch while potentially providing a useful system to help support the sales team.

The sales team frequently experienced difficulty with knowing if their prospective clients had read their proposals or not when sent by email. The Proposal Pusher was a simple platform that allowed the salespeople to upload their proposals and have it automatically email a link to the proposal to their prospective client. As the clients would need to follow the link to the Proposal Pusher website in order to download their proposal, this enabled the platform to track when a client had downloaded (and likely read) the proposal, and would send a Slack notification to the respective salesperson to give them a headsup to follow up on that prospective lead accordingly.

The Proposal Pusher was written and deployed over the course of 2 months, with continued improvements for the next 2 months. It was written in Ruby On Rails and deployed under the free tier offering of Heroku.