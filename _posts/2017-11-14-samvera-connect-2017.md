---
author: jfriesen
category: practices
filename: 2017-11-14-samvera-connect-2017.md
layout: post
tagline: Community-Driven Development
title: Samvera Connect 2017
tags: 'ruby, design, samvera'
---
Northwestern University hosted [Samvera Connect 2017][samvera_connect_2017] ([Connect's Planning Wiki Page][connect_planning_wiki_page]). Harsh P, LaRita R, Justin G, Jeremy F, Rick J, and Don B attended. Following the 3.5 day conference was 1.5 days of partner meetings ([agenda][partner_agenda] and [minutes][partner_minutes]).

## Report Backs

* Internal DLT Team Meeting
* [2017-11-14 DIS Department Meeting](https://wiki.nd.edu/pages/viewpage.action?title=2017-11-14+DIS+Program+Meeting&spaceKey=dlis)
* [2017-11-20 Sprint Planning / Retrospective document](https://docs.google.com/document/d/1SUtujkXBTTq8u_mWiumX6pyzT41Vl0MqpxZnkphfUVs/edit)

## Action Items

* Jeremy will join in the chartering of the [Core Components Working Group][core_components_charter]; In some way we'll participate
* Jeremy will begin chartering a "Permissions Refactor Working Group" with a goal to prototype a system for permission enforcement and grants.
* Harsh will draft a definition list of testing terms, draft an email, discuss, and promote document.
* Justin and Erin F of Stanford will draft an AWS Lambda best practices document and promote to the community.
* Justin will propose "Container Interest Group" with a goal of establishing and leveraging community best practices. *This is dependent on Jeremy proposing the Permissions Refactor Working Group and moving through that charter process.*
* Justin and Harsh will follow-up on why other institutions transitioned from CloudFormation to TerraForm.
* DLT will reconcile the community timeline with internal timelines for the Mellon Grant, Curate migration to Hyrax, and Hyrax adoption of Valkyrie.

## Notre Dame's Participation and Facilitation

* Panel participation for [Samvera and AWS](https://docs.google.com/presentation/d/1amcIx3YMEUyYn6ZedPblQiXkaK4QAlKE70F2poDR104/edit)
* Lightning Talk on [permission refactoring](https://docs.google.com/presentation/d/1U-ni8pAmGqRebKgQMCUNWclBOItcqbmAoxAHTEVF4Ys/edit)
* Lightning Talk on "[What I learned from working with students and how it’s positively changed our team](https://github.com/h-parekh/developer_notes/blob/master/What_I_learnt_from_working_with_student_developers.md)"
* Panel facilitation [What Should We Be Testing](https://docs.google.com/document/d/1Jg-6PWlb-jPztlXQV9a-zq6KVrbAZJUL_uJu9lkZ_9w/edit)
* Design discussion on [proposed permission refactoring][who_knows_what_can_can_can_can] (building on [previous notes](https://docs.google.com/document/d/1rUu1uBjAnNtGSIprQeEYkVLNFb6j5ioWJ7xNovE36Ns/edit))

## Sessions

See the [Google Drive Folder for detailed session notes][samvera_connect_session_notes]

### Key Takeaways

DLT team members believe that we should incorporate ESU into the Samvera community.

Other institutions are leveraging Docker, but this is an ad-hoc community effort. There are some minor changes we could submit to core gems to make container work easier.

This conference brought up distinction between Digital Collections (library digitization efforts) and Institutional Repository (capturing scholarly output), and the differentiation between these systems. *I believe we should be pivoting CurateND's effort to deliver on Digital Collections services.*

The governance groups began adapting Apache governance model's nouns and verbs. This was an organic result as we were describing processes, and does not reflect that we are adopting the Apache governance model.

There is repeated interest and curiosity in migrating Fedora 3 applications to Hyrax, while still leveraging Fedora 3. The pain point in migrating to Hyrax appears to be both an application migration and a data migration. The Hyrax w/ Fedora 3 migration path reduces one major moving part. We plan to explore that.

[IIIF](http://iiif.io/) has reached a saturation and maturation point.

Hyrax has externally facing APIs, though they are not adequately documented. We will want to further explore this. *I see [Archivo and Zotero integration in the routes](https://github.com/samvera/hyrax/blob/0aedcb8d5e668de26bdb87889149795b0d17897b/config/routes.rb#L195-L207). I also know we can [GET JSON-LD (`.jsonld`), N-Triples (`.nt`), and Turtle (`.ttl`) responses ](https://github.com/samvera/hyrax/blob/0aedcb8d5e668de26bdb87889149795b0d17897b/app/controllers/concerns/hyrax/works_controller_behavior.rb#L82-L90).*

#### Efforts of Other Institutions

Princeton (via [Figgy](https://github.com/pulibrary/figgy/)) and Lafayette are leveraging Samvera tools for book digitization workflows.

Other institutions, in particular Stanford, have years of experience in AWS. They are moving away from CloudFormation to TerraForm. What were they experiencing that we have not or are not yet experiencing?

[Documentation matters](https://medium.com/@oswebguy/why-documentation-matters-7152d46448e1). It is a means towards greater inclusion; Thank you to an expert developer for drawing attention to this. It helps people orient to the project and the community.

University of Michigan created a [GLOBUS](https://globus.org) connector in their Hyrax application to allow a repository user to asynchronously download a set of files to a GLOBUS location. This work was inspired by Notre Dame's work on Bendo and it's treatment of URLs and byte ranges.

University of Pennsylvannia has a similar storage topology as Notre Dame, there may be future alignment and conversations.

University of Michigan created [Fulcurm](https://www.fulcrum.org/)([source code](https://github.com/mlibrary/heliotrope)) for eBook publication and distribution.


## Partners Meeting

Discussion of Hyrax roadmap, spending, governance models, accessibility testing, documentation effort, and breakout session for roadmapping and resourcing.

### Key Takeaways

* Call for a working group to identify a community funded position (Technical Manager or Community Manager) by next partners meeting in March 2018.
* Breaking into smaller groups we discussed the goals and actions for governance, roadmapping, and resourcing the core components, solution bundles, and the overall community ([Session Breakout Notes][partner_breakout_activity]).
* Roadmap for Hyrax identified v3.0 for Valkyrie integration (currently on v2.0 release).
  - We (at ND) believe we can simplify application migration by creating a Fedora 3 Valkyrie adapter. Ben Armintor (of Columbia University) said we may be able to collaborate on that implementation.
* Core Components working group is forming (as of [2017-11-14 working on charter][core_components_charter])

[samvera_connect_2017]:https://nulib.github.io/samvera-connect2017/
[partner_breakout_activity]:https://docs.google.com/document/d/1gNAg8aTUhzAq5IXCDCmhX5l0HZXHK0mjCk2OdgI_jLM/edit#
[connect_planning_wiki_page]:https://wiki.duraspace.org/display/samvera/Samvera+Connect+2017
[partner_agenda]:https://wiki.duraspace.org/display/samvera/November+2017+Samvera+Partner+Meeting+Agenda
[partner_minutes]:https://docs.google.com/a/nd.edu/document/d/13FkiVsxrTYpMqa1d9exyeIdadn989vZoPDVuunMubYw/edit?usp=drive_web
[who_knows_what_can_can_can_can]:https://docs.google.com/document/d/12fq58AoVSyHknrG8ym-r-YZxyyQKZQGtqk-j1rCl3Hk/edit
[samvera_connect_session_notes]:https://drive.google.com/drive/folders/0Bzk4Z00sxBxXeDZsalhkcjBlTTA?usp=sharing
[core_components_charter]:https://docs.google.com/document/d/14A30SQ1CPpz6qP8c8KV0HiRGRVAatCE1rZ6qa3FsBLM/edit#
