---
title: "SchoolAir"
template: "content.html"
tags: [python, typescript, raspberrypi]
date: 2026-07-12
last_updated: 2026-10-14
desc: "Real-time air quality and data collection platform"
sub_title: "Capstone project with Alella Green Tech (AGT)"
github_url: "https://schoolair.org/en/blog.html"
under_construction: true
---

SchoolAir was a project I was fortunate to be part of during my final semester at RMIT University. It was a five-person industry capstone for Allela Green Tech (AGT), involving everything from Raspberry Pi sensors and local data storage to backend APIs, authentication and frontend services.

As project lead, I was responsible for coordinating the team, planning sprints, communicating with our industry partner and contributing to the backend architecture and implementation.

This write-up covers some of the engineering decisions I found most interesting, the challenges we encountered and what I'd do differently. Unfortunately, most of the software isn't open-source, so I've used simplified examples where I can't share the original code.

## 1. Background

Air quality can have significant effects on health and cognitive performance, particularly in environments such as classrooms. However, monitoring air quality isn't always straightforward for schools. Equipment can be expensive, and existing solutions may require technical expertise to configure and maintain.

SchoolAir aimed to make air-quality monitoring more accessible through inexpensive sensors and a central management platform.

An important consideration was data ownership and privacy. Schools needed control over their own information, while still having the option to contribute anonymised data to broader scientific research.

This introduced some interesting challenges around data isolation, user permissions and device management.

## 2. System overview

The system consisted of three main components:

* **Raspberry Pi sensors:** Python and SQLite applications responsible for collecting measurements, checking local alerts, retaining data and transmitting batches to the central server.

* **Central data server:** A TypeScript and Express REST API backed by PostgreSQL, responsible for ingestion, authentication, authorisation and managing organisations, schools, users and devices.

* **Frontend services:** Two React applications, one for platform-wide administration and another for organisations to manage their schools, devices and users.

Although the components were developed somewhat independently, they needed to work together reliably. A device might lose connectivity, a school might have multiple campuses, or several users might attempt to authenticate from the same network.

Much of our work involved anticipating these situations and deciding how the system should behave.

### 2.1 Organisations and access control

One of our early challenges was designing a user model that balanced flexibility with sensible access boundaries. Initially, we considered treating schools as the primary unit of ownership. However, this became limiting when considering organisations responsible for several schools or campuses. We eventually introduced an organisation layer, allowing multiple schools to belong to one organisation while maintaining boundaries between unrelated organisations. This provided a more natural structure for managing users, devices and permissions. It also gave us a foundation for role-based access control, where a user's ability to perform an action depended on their role and the resources they belonged to.

A lesson from this process was the importance of getting the underlying data model right and trying to anticipate possible use cases *before* building too much around it. Small decisions about ownership can have huge consequences throughout an entire system. If we had stuck with schools, we would have had a painful refactor on our hands.

### 2.2 Reliable data collection

For the Raspberry Pi application, we needed a way to preserve sensor measurements during connectivity interruptions. I chose SQLite for local storage rather than keeping everything in memory or introducing a separate database server such as MySQL.

SQLite offered a good balance of simplicity, durability and low resource usage. It also meant we could store measurements locally without depending on a network connection or managing another service. Measurements were queued locally and transmitted to the central server in batches. This allowed devices to recover from temporary outages without immediately losing their collected data. We also introduced configurable collection schedules to reduce unnecessary transmissions. During school hours, measurements could be reported more frequently, while outside these hours the reporting interval could be reduced. Local aggregation and retention policies helped manage storage growth over time.

The main trade-off was increased complexity on the device. We now had to consider queue management, retry behaviour and data retention, rather than simply sending each measurement as it arrived. However, these are all important features of this type of IoT system and needed to be implemented with care.

### 2.3 Protecting the central API

Another concern was protecting the central server from excessive requests and incorrectly configured or potentially compromised devices.

For sensor ingestion, we introduced limits on request frequency and payload size. These provided some protection against devices overwhelming the server through unusually frequent or oversized transmissions. Authentication required a slightly different approach. In a school environment, many users may share a single public IP address. Aggressive IP-based rate limiting could therefore prevent legitimate users from signing in.

We used a relatively generous authentication limit and excluded successful requests from the count:

```js
export const authLimiter = rateLimit({
    windowMs: FIFTEEN_MINUTES_IN_MILLISECONDS,
    limit: 100,
    standardHeaders: true,
    legacyHeaders: false,
    skipSuccessfulRequests: true,
    message: {
        error: "Too many authentication attempts, please try again later."
    },
});
```

We recognise that this wasn't a perfect solution. IP-based rate limiting is a coarse mechanism, particularly when many users share a network. However, we decided that it was a reasonable safeguard within the project's scope.

We also recommended additional traffic filtering at the hosting or infrastructure level, allowing abusive requests to be intercepted before reaching the application; which is preferable for this type of safety measure anyway.

## 3. Testing and integration

Although we used generated data to populate parts of the frontend, much of our integration testing involved running simulated Raspberry Pi devices with mock sensor readings.

```bash
ofloeck@jackal:~/Git/agt-rmit-schoolair/pi main
$ systemctl --user status schoolair-dev
● schoolair-dev.service — SchoolAir sensor ingest (DEV)
     Loaded: loaded (/home/of oeck/.config/systemd/user/schoolair-dev.service; disabled; preset: enabled)
     Active: active (running) since Sun 2026-06-14 17:55:22 AEST; 7h ago
   Main PID: 6751 (python)
      Tasks: 2 (limit: 9135)
     Memory: 39.4M (peak: 45.8M)
        CPU: 11.00s
     CGroup: /user.slice/user-1000.slice/user@1000.service/app.slice/schoolair-dev.service
             └─6751 /home/of oeck/Git/agt-rmit-schoolair/pi/venv/bin/python -u main.py

Jun 15 01:05:43 jackal python[6751]: Queue large (160 pending) — alert checking skipped
Jun 15 01:05:43 jackal python[6751]: Draining 160 measurement(s)...
Jun 15 01:05:43 jackal python[6751]: Lost connection during drain — will retry next cycle
Jun 15 01:05:43 jackal python[6751]: 160 measurement(s) still queued — server may be of ine
Jun 15 01:05:43 jackal python[6751]: [idle] next reading in 300s

Jun 15 01:10:43 jackal python[6751]: Queue large (161 pending) — alert checking skipped
Jun 15 01:10:43 jackal python[6751]: Draining 161 measurement(s)...
Jun 15 01:10:43 jackal python[6751]: Lost connection during drain — will retry next cycle
Jun 15 01:10:43 jackal python[6751]: 161 measurement(s) still queued — server may be of ine
Jun 15 01:10:43 jackal python[6751]: [idle] next reading in 300s
```

This allowed us to exercise the system end-to-end: registering a device, collecting measurements, transmitting data, storing it centrally and verifying alert behaviour. Testing complete workflows was particularly useful because many issues only became apparent when individually working components were connected. It also reinforced the value of designing systems that can be tested without depending on physical hardware being available.

## 4. What I'd do differently

Looking back, there are several things I'd approach differently.

* **Design for unreliable connections earlier:** Reliable transmission, batching and recovery behaviour deserved more attention during our initial architecture discussions. These are fundamental requirements for distributed sensor systems, rather than features to consider later.

* **Simplify configuration management:** Keeping device configuration synchronised with the central server introduced challenges of its own. With more time, I'd investigate a clearer configuration versioning and acknowledgement mechanism to make updates more predictable.

* **Provide more implementation context during sprint planning:** As project lead, I learned that assigning a task isn't always enough. Providing examples, expected behaviour and clear acceptance criteria can prevent misunderstandings and reduce unnecessary rework.

* **Document decisions as they happen:** Because different team members were responsible for different parts of the system, good documentation was essential. I'd make a greater effort to record architectural decisions and their reasoning throughout development, rather than relying on knowledge held by individual developers.

## 5. Final thoughts

SchoolAir was an excellent opportunity to experience the challenges of developing a system with several interconnected components and a real industry partner.

Beyond the technical work, I learned a great deal about coordinating people, communicating requirements and making engineering decisions within real constraints.

I'm particularly grateful to my teammates and the people at Allela Green Tech for their support throughout the project.

It was a rewarding experience, and one that gave me a much better appreciation for the engineering and communication involved in building software that needs to work reliably outside a development environment.