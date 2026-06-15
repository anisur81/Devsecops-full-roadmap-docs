```markdown
# Basics of DevOps

## What is DevOps

DevOps is a combination of two functions that are typically treated separately: development and operations. That definition of DevOps is the *what*. The *why* is even easier.

Standardized development methodology, clear communication, and documented processes supported by a standards-based, proven middleware platform improve application development and management cycles, bring agility, and provide greater availability and security to your IT infrastructure.

Clearly, DevOps is about connecting people, products, and processes. Ultimately, DevOps is about connecting IT to business.

By fostering collaboration and leveraging automation technologies, DevOps enables faster, more reliable code deployment to production in an efficient and repeatable manner.

### The Main Goal of DevOps

- Faster software delivery
- Continuous integration and deployment
- Better software quality
- Automation of infrastructure and workflows
- Improved collaboration and communication

## Evolution of DevOps

Before DevOps became common practice, development and operations existed as two separate components of the application release process. The development team would write code and pass it to the operations team, who would then deploy it to the production environment.

Part of the problem with this process is the tension between what the developers are trying to achieve versus the goals of the operations team.

Application development pushes for continuous updates to add features, fix defects and make other necessary changes, often as fast as possible. IT operations, on the other hand, tries to limit the number of releases as they are responsible for running a stable system with the highest percentage of uptime.

To further complicate matters, there was often no clearly defined or automated process for handing over applications. This would result in miscommunications and misalignments, such as:

- Developers writing code without considering where or how the code would be deployed or passing along poorly-documented deployment guides.
- Operations not fully understanding what they were deploying and why, resulting in inefficiencies.
- Additionally, any time they would discover issues with the application they would have to send it back to the developers with improvement suggestions.

## DevOps Lifecycle

The DevOps Lifecycle divides the SDLC lifecycle into the following stages:

*(Diagram placeholder - refer to original visual)*

## 7 Cs of DevOps

1. Continuous Development
2. Continuous Integration
3. Continuous Testing
4. Continuous Deployment/Continuous Delivery
5. Continuous Monitoring
6. Continuous Feedback
7. Continuous Operations

### 1. Continuous Development

This step is crucial in defining the vision for the entire software development process. It focuses mostly on project planning and coding. At this phase, stakeholders and project needs are gathered and discussed. In addition, the product backlog is maintained based on customer feedback and is divided down into smaller releases and milestones to facilitate continuous software development.

Once the team reaches consensus on the business requirements, the development team begins coding to meet those objectives. It is an ongoing procedure in which developers are obliged to code whenever there are modifications to the project requirements or performance difficulties.

### 2. Continuous Integration

Continuous integration is the most important stage of the DevOps lifecycle. At this phase, updated code or new functionality and features are developed and incorporated into the existing code. In addition, defects are spotted and recognized in the code at each level of unit testing during this phase, and the source code is updated accordingly. This stage transforms integration into a continuous process in which code is tested before each commit. In addition, the necessary tests are planned during this period.

### 3. Continuous Testing

Some teams conduct the continuous testing phase prior to integration, whereas others conduct it after integration. Using Docker containers, quality analysts regularly test the software for defects and issues during this phase. In the event of a bug or error, the code is returned to the integration phase for correction. Moreover, automation testing minimises the time and effort required to get reliable findings. During this stage, teams use technologies like Selenium. In addition, continuous testing improves the test assessment report and reduces the cost of delivering and maintaining test environments.

### 4. Continuous Deployment

This is the most important and active step of the DevOps lifecycle, during which the finished code is released to production servers. Continuous deployment involves configuration management to ensure the proper and smooth deployment of code on servers. Throughout the production phase, development teams deliver code to servers and schedule upgrades for servers, maintaining consistent configurations.

In addition to facilitating deployment, containerization tools ensure consistency throughout the development, testing, production, and staging environments. This methodology enabled the constant release of new features in production.

### 5. Continuous Feedback

Constant feedback was implemented to assess and enhance the application's source code. During this phase, client behaviour is routinely examined for each release in an effort to enhance future releases and deployments. Companies can collect feedback using either a structured or unstructured strategy.

Under the structural method, input is gathered using questionnaires and surveys. In contrast, feedback is received in an unstructured manner via social media platforms. This phase is critical for making continuous delivery possible in order to release a better version of the program.

### 6. Continuous Monitoring

During this phase, the functioning and features of the application are regularly monitored to detect system faults such as low memory or a non-reachable server. This procedure enables the IT staff to swiftly detect app performance issues and their underlying causes. Whenever IT teams discover a serious issue, the application goes through the complete DevOps cycle again to determine a solution. During this phase, however, security vulnerabilities can be recognized and corrected automatically.

### 7. Continuous Operations

The final phase of the DevOps lifecycle is essential for minimising scheduled maintenance and other planned downtime. Typically, developers are forced to take the server offline in order to perform updates, which increases the downtime and could cost the organisation a large amount of money. Eventually, continuous operation automates the app's startup and subsequent upgrades. It eliminates downtime using container management platforms such as Kubernetes and Docker.

## Additional Phases for DevSecOps

DevSecOps is the integration of security into a pipeline of continuous integration, continuous delivery, and continuous deployment. By infusing DevOps values into software security, security verification becomes an integral, active component of the development process.

Similar to DevOps, DevSecOps is an organisational and technological paradigm that blends automated IT technologies with project management workflows. DevSecOps incorporates active security audits and security testing into agile development and DevOps workflows so that security is incorporated into the product, as opposed to being added after the fact.

### For teams to deploy DevSecOps, they must:

- Incorporate security throughout the software development lifecycle to decrease software code vulnerabilities.
- Ensure that the whole DevOps team, including developers and operational teams, is liable for adhering to security best practices.
- Facilitate automatic security checks at each level of software delivery by integrating security controls, tools, and processes into the DevOps workflow.

## DevSecOps Lifecycle

### Plan

The plan phase is the least automated part of DevSecOps, requiring security analysis collaboration, debate, review, and strategy.

Teams should conduct a security analysis and develop a plan outlining where, when, and how security testing will be conducted. IriusRisk, a collaborative design tool for threat modelling, is a prominent DevSecOps planning tool. Issue tracking and management systems such as Jira Software and communication and discussion technologies such as Slack are further tools.

### Build

As developers contribute code to the source repository, the build step begins. DevSecOps build technologies prioritise automating the security examination of build output artefacts. Software component analysis, static application software testing (SAST), and unit tests are essential security approaches. An existing CI/CD pipeline can be augmented with tools for automating these tests.

Installing and building upon third-party code dependencies, which may be from an unknown or untrusted source, is common practice for developers. External code dependencies may mistakenly or maliciously incorporate vulnerabilities and exploits. It is essential to review and scan these dependencies for security vulnerabilities during the build phase.

**Well-known analysis tools during the build step:** OWASP Dependency-Check, SonarQube, SourceClear, Retire.js, Checkmarx, and Snyk.

### Code Phase

The code phase DevSecOps tools assist developers in writing better secure code. Key security procedures throughout the code-phase include static code analysis, code reviews, and pre-commit hooks.

When security technologies are directly integrated into the Git workflow of developers, every commit and merge will automatically trigger a security test or review. Several programming languages and integrated development environments are supported by these technologies. Gerrit, Phabricator, SpotBugs, PMD, CheckStyle, and Detect Security Bugs are a few of the most prominent security code tools.

### Test

After a build artefact has been successfully delivered to staging or testing environments, the test phase is launched. Running a complete test suite requires substantial time. This step should fail quickly so that the more costly testing tasks can be performed last.

The testing phase employs dynamic application security testing (DAST) instruments to detect real application processes such as user authentication, authorisation, SQL injection, and API-related endpoints. The security-focused DAST compares an application against a list of known critical vulnerabilities, such as the OWASP Top 10.

**Available open source and paid testing solutions:** BDD Automated Security Tests, JBroFuzz, Boofuzz, OWASP ZAP, Arachi, IBM AppScan, GAUNTLT, and SecApp suite.

> **Pro Tip:** DevOps and DevSecOps cannot be properly implemented without automated real device testing. Teams must make substantial use of automated tools for constructing, testing, reviewing, deploying, and monitoring code in order to maintain the deployment frequency attained by these techniques.

DevSecOps requires a collection of security testing tools (or tools that additionally cover security modules) in addition to the CI/CD technologies that are necessary for DevOps success.

BrowserStack integrates with various CI/CD tools to facilitate the implementation of DevOps. This includes Jira, Jenkins, TeamCity, Travis CI, and more technologies. In addition, it provides a cloud Selenium grid of over 3000 genuine browsers and devices for testing. Moreover, built-in debugging tools allow testers to instantly detect and address errors.

### Deploy

If the preceding processes pass, it is time to deploy the build artifact to production. During the deployment phase, the security concerns to address are those that only occur against the live production system. For instance, all configuration variations between the production environment, the prior staging environment, and the development environment should be properly evaluated. The TLS and DRM certificates used in production should be checked and reviewed for impending renewal.

The deploy step is an ideal time for runtime verification tools such as Osquery, Falco, and Tripwire, which gather information from a running system to verify if it is performing as expected. Companies can also apply chaos engineering ideas by conducting experiments on a system to increase confidence in the system's ability to endure chaotic situations. It is possible to replicate real-world occurrences such as server crashes, hard disk failures, and lost network connections. Netflix's Chaos Monkey tool, which employs chaotic engineering ideas, is well-known. Moreover, Netflix has a Security Monkey application that searches for vulnerabilities or violations in badly configured infrastructure security groups and disables susceptible servers.

### Release Phase

By the time the DevSecOps cycle reaches the release phase, the application code and executable should have been properly tested. The phase focuses on safeguarding the infrastructure of the runtime environment by reviewing configuration values for the environment, such as user access control, network firewall access, and secret data management.

### Observe

Further security precautions are required once an application has been deployed and stabilised in a production environment. Using automated security checks and security monitoring loops, businesses must monitor and observe the live application for any attacks or breaches.

Runtime application self-protection (RASP) discovers and stops inbound security threats automatically and in real-time. RASP functions as a reverse proxy that monitors incoming threats and enables the application to change itself automatically in response to defined conditions without requiring human participation.

A specialised internal or external team can conduct penetration testing to identify exploits or vulnerabilities by compromising a system on purpose. Offering a bug bounty program that compensates external individuals who reveal security exploits and flaws is another security measure.

Using analytics, security monitoring instruments monitor crucial security-related variables. These tools, for instance, mark requests to sensitive public endpoints, such as user account access forms and database endpoints. Popular runtime defensive instruments include Imperva RASP, Alert Logic, and Halo.
```

You can copy this directly into a `.md` file and it will render correctly in any Markdown viewer (GitHub, GitLab, VS Code, etc.).
