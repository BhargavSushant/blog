---
title: "About"
layout: "single"
url: "/about/"
summary: "Sushant Bhargav, software engineer, infrastructure, research"
ShowReadingTime: false
ShowBreadCrumbs: false
ShowToc: false
---

I am a software engineer in Bhopal, India. About fifteen years in, split fairly evenly between building software and running the infrastructure it sits on. Most of that time I have been somewhere in the middle: close enough to the application to know why it is slow, close enough to the machine to know why it fell over.

## Software development

Backend and full-stack work, mostly JavaScript and TypeScript. Node.js, Express, ReactJS, Redux, MongoDB, MySQL. Java and Python where the job called for them.

The work I would point to: breaking a monolithic notification system into microservices with real service boundaries and API contracts for a North American B2C platform, including Kafka-driven flows across inventory, order lifecycle, refunds and returns. Modernising a legacy .NET portal into ReactJS micro-frontends for three regions, each with its own taxation, branding and compliance rules. Before that, a decade of product delivery: eCommerce platforms, listing portals, and Android applications for government agencies that reached over 50,000 users.

I care about the unglamorous parts. Refactoring for clarity. Code review that catches design problems rather than style ones. Debugging the class of bug that only shows up under load.

## Infrastructure

This is where my depth actually is, and it took me a while to say so plainly.

Eight years of hands-on Linux. Servers, hosting, deployment, systemd, process-level debugging. The kind of knowledge that comes from things breaking at inconvenient hours rather than from a lab exercise. Containerised deployments with Docker, CI/CD through Jenkins, GitHub Actions and Travis CI, provisioning on AWS and GCP.

What interests me now is infrastructure as a system with state: provisioning that is reproducible, changes that are transactional, rollback that actually rolls back. Which is a longer way of saying I have spent enough time cleaning up after configuration management to have opinions about it.

Currently building a Linux hardening toolkit with real transactional semantics. A typed state manifest, drift detection against effective runtime state rather than file hashes, and an explicit register of controls that cannot be reversed without a reboot. Provisioned with Terraform, tested across VM images in CI. I am writing about it as it goes.

## Independent work

Since late 2024 I have worked as an independent contractor rather than an employee. Currently with Girnarsoft Education Services, where I deliver technical training on Terraform, GCP, cloud infrastructure, JavaScript and algorithm design to students and faculty, and consult on cloud adoption for institutions. Over twenty faculty development programmes so far.

I have also built AI-assisted curriculum tooling there: Python and Node services with prompt-based generation, validation and human-in-the-loop review, deployed on GCP.

Alongside that I coordinate a computing programme at SAGE University, Bhopal, covering Data Science, full-stack development and related subjects. A good deal of my automation work exists because academic administration generates an enormous volume of documents and I refuse to produce them by hand.

## Research

I am pursuing a PhD, with a research interest in quantum computing. Currently at the foundations rather than anywhere near the frontier, and honest about the distance.

I am also seriously interested in machine learning, though more in the operational half than the modelling half. MLOps, reproducibility, pipeline orchestration, and the resource allocation problems that show up when ML systems meet real infrastructure budgets. That sits naturally next to the infrastructure work, since most of the hard problems in production ML turn out to be infrastructure problems wearing a different hat.

## What I write about here

Linux hardening, and the recurring problem of configuration that looks correct and does nothing. Infrastructure automation, state and rollback. Occasionally the places where development and operations disagree about what "done" means.

Most posts start from something that broke, or nearly did.

---

**Elsewhere:** [GitHub](https://github.com/BhargavSushant) · [LinkedIn](https://www.linkedin.com/in/sushantbhargav/) · [BhargavSushant@gmail.com](mailto:BhargavSushant@gmail.com)

Open to remote infrastructure and platform engineering work.
