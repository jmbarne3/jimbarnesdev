---
title: How to Foster a Development Dream Team
description: Practical approaches to fostering a successful and happy development team.
date: 2023-06-19 06:00:00 -0400
categories: [Opinion, Teams]
tags: [communications, team-building, code review, pull requests]
img_path: /assets/images/posts/fostering-dev-dream-team/
image:
  path: hero.jpg
  alt:
---

An important&ndash; and often overlooked &ndash;aspect of any developer's set of skills is their ability to interact and work well with others. Throughout my career, I've worked on several teams that consistently exceeded the expectations of management and served as a model for excellence across other departments. While these teams were made up of very competent individuals, it was the culture within the team that contributed most to their success.

Fostering a healthy culture for your team can be a force multiplier, creating efficiencies around the interoperability of code, documentation and troubleshooting. In my opinion, fostering that healthy culture starts with each individual on the team. In my personal experience, systems put into place by management to try to create team cohesion seldom succeed. It's when developers self-govern around principles of standardization, internal accountability and recognition, and a sense of ownership that they are most likely to succeed.

Today I want to talk about how to foster that culture through some concrete practices you can put into place.

## Create Clear Expectations

It's difficult to succeed in any situation where the end goals and objectives are not made clear. Developing and **writing down** expectations not only makes onboarding new developers much easier but gives the entire team something to refer to when guidance is needed. When many developers start in a new role, they may not immediately be comfortable asking for help. Having expectations documented provides a guide they can refer to directly for the majority of issues they may come across.

I will cover three areas of documentation below that have been invaluable to the teams I've worked on throughout my career.

### Create a Style Guide

A style guide should document a few things about how your team writes code:

1. If there are syntactical variations in the way common blocks of code can be written, the style guide should specify how the code should be written. For example, we still use [PEP 8](https://peps.python.org/pep-0008/) standards for Python projects and [WordPress Coding Standards](https://developer.wordpress.org/coding-standards/wordpress-coding-standards/) for WordPress projects. You don't always have to create a guide of your own. If the project you're working on uses a framework with a style guide, you can use that!
2. If there are various ways in which projects can be organized, this should be specified. This is a common problem within the world of Python development, as projects can be organized in various ways and can use different package management systems. Which organizational pattern you use is less important than having a documented standard and following it on every project.
3. If your project uses multiple languages and frameworks (e.g. Python Django with an Angular frontend), then the project should specify the style guides for _each language and framework_. Unless you have specific styles for when you use Angular within a Django project, then you will probably have separate style guides for both. Just make sure to link to each one within the project's readme or contributing files.

Most importantly, once you've agreed upon and developed your style guides, _follow them_! The style guide isn't just for new or junior developers, it's for everyone on the team. One way to encourage use is to use linters for popular style guides, such as [Pylint](https://marketplace.visualstudio.com/items?itemName=ms-python.pylint) or [flake8](https://marketplace.visualstudio.com/items?itemName=ms-python.flake8) for PEP 8 standards. Encourage your team to use tools that enforce the style guide.

### Create Documentation Around Code Reviews

If you host your code on GitHub, there are a number of features available that can help set clear expectations about what is expected before code is reviewed. The simplest feature is writing a `CONTRIBUTING.md` file and adding it to the root of your repository. When a new contributor attempts to create a pull request for the first time, they will be encouraged to read the `CONTRIBUTING.md` file with a pop-up in the pull request UI.

Use this space to set expectations around what is acceptable code and not. If you're following one or more style guides in your project, the `CONTRIBUTING.md` file is an excellent place to document this. In addition, if there are any policies around testing or other processes before code is reviewed, they can be documented here.

### Create a Pull Request Template

Another useful feature of GitHub is the `pull_request_template.md` file. This file can be stored in the `.github` directory of your project and will provide the default text within the description field when a new pull request is created. You can treat the template like a form, creating areas to be filled out by the developer creating the pull request.

```markdown
<!---
Thank you for contributing to the { Project Name }.

Provide a general summary of your changes in the Title above and fill in the template below.
-->

**Description**
<!--- Describe your changes in detail -->

**Motivation and Context**
<!--- Why is this change required? What problem does it solve? -->
<!--- If it fixes an open issue, please link to the issue here. -->

**How Has This Been Tested?**
<!--- Please describe how you tested your changes. -->

**Types of changes**
<!--- What types of changes does your code introduce? Put an `x` in all the boxes that apply: -->
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to change)

**Checklist:**
<!--- Go over all the following points, and put an `x` in all the boxes that apply. -->
<!--- If you're unsure about any of these, don't hesitate to ask. We're here to help! -->
- [ ] My code follows the code style of this project.
- [ ] My change requires an update to the documentation.
- [ ] I have updated the documentation accordingly.

```
{: file="pull_request_template.md" }

This encourages a consistent format in the way pull requests are created and sets the expectations what should be done prior to submitting the request. For example, in the template above, developers are expected to check the "My change requires an update to the documentation" box if that's the case. This is a subtle encouragement to the developer to ensure they've updated or at least plan to update the documentation prior to releasing the code.

## Constructive Code Reviews

The first section takes for granted this second, extremely important point: developers should review each other's code. Code reviews accomplish a few things when done well:

1. Everyone on the team sees every line of code. This allows for continual cross-domain training and knowledge transfer as just a normal part of the code writing process.
2. Code reviews allow for the building of trust between teammates. Having your code reviewed is usually one of the most uncomfortable things that new developers experience. Making this experience constructive and encouraging is a simple and highly effective way to build confidence and trust among newer developers.
3. Code reviews create accountability. When you know every line of your code is going to be looked at by your team members, you are highly motivated to write good code.

Code reviews done well can have a huge impact on the morale and competency of your team. However, code reviews done badly can have an equally negative impact on those aspects of your team. Let's take a look at a few things we can do to ensure our code reviews are effective.

### Keep it Positive

The ultimate purpose of code review for most teams is to ensure only quality code makes it into the production code base. This necessarily makes it a critical exercise. However, being critical doesn't necessarily mean being _negative_.

One excellent way to accomplish this is to point out areas where the code is well-written or new to you as the reviewer. If your colleague uses a function you've never seen before, point it out! If they developed a particularly elegant solution to a problem, praise them for their ingenuity and creativity. This is not only a positive benefit for the person whose code is being reviewed, but can sometimes help highlight a particular approach for others reviewing the code, furthering the knowledge transfer.

When code needs to be corrected or improved, point it out with reasons and suggestions. If a developer has written an inefficient function, point out what causes the inefficiency and provide an example of how to work around it. Simply pointing out it's inefficient does not meet the objective of the code review: to get the best code into production.

### Keep it Short

One way to encourage constructive code reviews is the make sure the change set being reviewed is small. Try to remember the people reviewing code also have their own things to do. Sometimes, it's simply unavoidable, especially when starting new projects if there is a lot of boilerplate. However, for changes to existing code bases, keeping the change set small and focused on a particular feature or fix is an excellent way to get the best feedback from your team members.

### Create a Rhythm

A fantastic way to encourage good reviews is to have them happen at a regular time. Together with keeping reviews short, a daily rhythm can be established where new code is written or completed in the morning, reviewed in the afternoon, and then merged and prepared for release at the end of the day. This schedule might not work for every team or organization, but having some sort of regular rhythm can encourage good reviews by allowing everyone to have a set time when they go over reviews. In theory, they should not have to interrupt their own work to do the reviews as their own code will be up for review during that time as well.

### Use Your Team's Strengths

On many teams I've been on, different developers have different strengths. I've generally covered backend and database-related code and I've worked with some very talented frontend developers. I relied heavily on them when writing frontend code to keep me on track, and they relied heavily on me when it came to backend code. While we all wrote for the entire stack, we recognized we all had strengths and weaknesses in our skill sets and supported each other through the code review process.

And that's really what all this comes down to: code review is for _supporting_ your fellow developers, not for inflating your ego. Use your skills and abilities to build up your team members and encourage them to grow in their abilities and talents.

## Communicate
Clear communication between team members is a must-have. All the documentation we've discussed so far is a form of communication. If developed as a team, it's democratic and becomes institutional through adoption and practice.

However, it is also important to communicate regularly outside of these channels. If your organization uses Slack or Teams, create a channel for your team to collaborate on. If possible, create separate channels for project communication and team communication. In the same way, you create separation of concerns within a development project, having channels dedicated to specific types of communication can be extremely helpful in allowing team members to focus while allowing for a free space for discussion.

### Communicate Transparently

One approach that I have found particularly useful throughout my career is communicating openly on projects. If a discussion needs to happen about a project or code that's being reviewed, have that discussion in a public channel. As a team, become comfortable with having those discussions publicly. Not only is this an issue of accountability, but it also normalizes healthy behavior among teammates.

If you need help writing a bit of code or understanding how something works, discuss it publicly. This not only allows the knowledge to be seen and consumed by everyone else on the team, but it also removes the stigma behind asking for and needing help. Nobody on the team knows everything and everyone will need to ask for help at some point. Make it a habit to do this on public channels.

### Have a Private Space

That said, having a private space for your immediate team is also important. On past teams, we've used private messenger groups or Slack channels. Use this space to make lunch plans, talk about video games, sports, or the crazy Florida weather! Use this space to just be people talking to other people about interesting things. Do _not_ use this space to talk about work. You have public channels you can do that in - this space is a work-free space, a virtual water cooler.

## Final Thoughts

All of the above points really come down to two primary things:

1. Set and clearly communicate realistic expectations
2. Be a supportive and decent human being to your teammates

There are numerous ways to do those two things, and what's covered above only scratches the surface of the plethora of ways to approach working with other people. However, the approaches described above create a strong foundation on which to build. They create an environment in which teammates _want_ to meet expectations and contribute to the team's collective success.
