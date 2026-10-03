---
title: "About"
description: Dima Mikielewicz. A little story about me.
draft: false
---

## Social

- [GitHub](https://github.com/dimamik)
- [LinkedIn](https://www.linkedin.com/in/dima-mikielewicz-5786961b4/)
- [Twitter/X](https://x.com/dimamikielewicz)
- [Bluesky](https://bsky.app/profile/dimamik.bsky.social)
- contact@dimamik.com

## About

I'm a Software Engineer with 8 years of experience building production Elixir systems and, more recently, real-time AI
agents in Python and Elixir. I've worked as a senior IC across several teams at once and as a tech lead taking a
team from idea to a shipped product.

I like building elegant solutions to complex problems. Let's chat!

## Experience

### [Software Mansion](https://swmansion.com/) - Software Engineer

**[Feb 2022 - present]** Contracting for [dscout](https://dscout.com), a US user-research platform with an Elixir/Python/React stack **(Phoenix, PostgreSQL, Oban, Redis, GraphQL)**:

- Led a team of 5 engineers
- Helped build AI Mod, a real-time voice AI moderator **(Pipecat, Elixir, React)**
- Migrated the entire application from Ruby on Rails to Elixir; the codebase is now 1.4 million lines of Elixir, one of the largest Elixir codebases in production
- Created an internal data migrations framework and designed and built the audit log system
- Led the effort to streamline the primary data source for participant incentives: simpler model, lower costs
- Worked on permissions and access control
- Created [Learn Elixir](https://github.com/dimamik/learn_elixir), the internal onboarding program for engineers new to Elixir; mentored and code-reviewed **10+** people
- [Legion](https://github.com/software-mansion/legion), my open-source AI-agent framework, is now developed under Software Mansion
- Speaker: ElixirConf EU 2026 ("Legion: Agentic Code Execution with Pure Elixir"), Elixir Meetup Chicago 2026, Code BEAM Europe in Stockholm (Torus)

**[2022]** Before dscout, a few months on the [Membrane Framework](https://membrane.stream/) team, Software Mansion's multimedia framework in Elixir. I gave the introduction to Membrane talk at a Software Mansion meetup:

  <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;">
    <iframe 
      src="https://www.youtube.com/embed/9ngSYB8QVNE?si=hvlScENgu8DcgBp1" 
      title="YouTube video player"
      frameborder="0" 
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
      referrerpolicy="strict-origin-when-cross-origin"
      allowfullscreen
      style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"
    ></iframe>
  </div>

### Open Source Author

**[Dec 2024 - present]** Author and maintainer of Elixir libraries for AI agents, durability and search:

- [Legion](https://github.com/software-mansion/legion) - framework for AI agents that generate and execute sandboxed Elixir code instead of tool-calling. Fewer LLM round-trips, multi-agent orchestration, human-in-the-loop, structured output, telemetry. Started as a personal project, now developed under Software Mansion, with the legion_web companion package. Presented at ElixirConf EU 2026.
  <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;">
    <iframe 
      src="https://www.youtube.com/embed/VrtoRuBCWBc?si=zf5K98CgOw9V-BVW" 
      title="YouTube video player"
      frameborder="0" 
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
      referrerpolicy="strict-origin-when-cross-origin"
      allowfullscreen
      style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"
    ></iframe>
  </div>

- [Torus](https://github.com/dimamik/torus) - PostgreSQL full-text, BM25, similarity and semantic search for Ecto. Presented at Code BEAM Europe in Stockholm. Try it out at [torus.dimamik.com](https://torus.dimamik.com).
  <div style="position: relative; padding-bottom: 56.25%; height: 0; overflow: hidden; max-width: 100%;">
    <iframe 
      src="https://www.youtube.com/embed/T_B8lQh_f4Q?si=j2UPMu_9xR0AJov_" 
      title="YouTube video player"
      frameborder="0" 
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
      referrerpolicy="strict-origin-when-cross-origin"
      allowfullscreen
      style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;"
    ></iframe>
  </div>

- [Revenant](https://github.com/dimamik/revenant) - durable GenServers backed by Postgres.
- [Vault](https://github.com/dimamik/vault) - lightweight immutable data storage within a process subtree.
- [RuleBook](https://github.com/dimamik/rule_book) - deterministic forward-chaining rules engine.
- [Feline](https://github.com/dimamik/feline) - real-time voice and multimodal AI pipelines for Elixir, inspired by Pipecat.
- Contributions to Phoenix, Credo, Oban ([oban-py](https://github.com/oban-bg/oban-py)) and Membrane (camera capture, WAV, FLAC and raw-audio plugins, Beamchmark).

### Board Member - BIT Student Scientific Group at AGH University

**[Aug 2021 - Jul 2022]**

- Ran introductory web development courses for students
- Planned the group's programme: decided which sections and topics best fit students' needs and level

### Freelance - Software Engineer

**[May 2018 - Sep 2021]**

- Recommendation system for a large literary portal: personalized book feed built from each reader's interactions **(Python, pandas, NumPy, scikit-learn, Flask)**. Covered the full pipeline:
  - analyzed reading behavior and engagement data, and cleaned and aggregated interaction logs
  - engineered features from user activity and book metadata (genres, authors, text descriptions via TF-IDF)
  - combined collaborative filtering with content-based similarity, with fallbacks for new users and new books (cold start)
  - evaluated models offline (precision@k, recall@k) before rollout
  - served recommendations through a Flask API with periodic retraining
  - [A Proof Of Concept that I used to recruit myself](https://github.com/dimamik/what-to-read)
- Web applications for local businesses **(Django, FastAPI, NestJS, PostgreSQL)**

## Side projects

- [Live Piano](https://piano.dimamik.com) - play your MIDI keyboard (or virtual piano) for your friends, live, directly in the browser. Uses WebRTC for low-latency streaming of MIDI events.
- [Live Chess](https://chess.dimamik.com) - play chess with your friends.
- [Advent of Code 2024](https://github.com/dimamik/advent_of_code_2024) - 25 programming languages, each day in a new programming language.
- [Subtitles player](https://github.com/dimamik/subtitles-player) - takes a video, generates subtitles, and plays the video alongside the generated subtitles.
- [Schedule your life](https://github.com/agh-kiwis/agh-kiwis) - uses time allocation algorithms to chunk and plan long-term activities throughout your calendar. Reallocates events if they are not completed on time.
- [stateofelixir.com](https://stateofelixir.com) - a PoC, similar to [State of JS](https://stateofjs.com) and [State of React Native](https://stateofreactnative.com), but for Elixir and built with Elixir - [GitHub repo](https://github.com/dimamik/state_of_elixir). The project is on hold since there were other surveys at the same time, and I answered most of the questions I had when doing a PoC and gathering questions.
- And a few others, not yet publicly shared.

## Education

- **[2019 - 2022]** Bachelor of Engineering in Computer Science at [AGH University of Science and Technology](https://www.agh.edu.pl/en/)
