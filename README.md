# HTTP/2 Dialogues

HTTP/2 Dialogues is a collection of learning materials about HTTP/2 programming and architecture.

The goal of this repository is to make HTTP/2 easier to understand for programmers by explaining its concepts through dialogue.

Rather than presenting the protocol as a dense specification, this project explores HTTP/2 step by step through conversations about how it works, how it appears in programming APIs, and why it was designed the way it is.

## Characters

Articles in this repository are written as conversations between three roles.

**Learner**  
Represents the reader's questions when encountering HTTP/2 concepts for the first time.

**Engineer**  
Explains how HTTP/2 concepts appear in real implementations and libraries.

**Architect**  
Explains the protocol design and architectural reasoning behind HTTP/2.

This structure connects three perspectives:

- what developers are confused about
- how implementations actually work
- why the protocol is designed this way

## Purpose

This repository aims to help programmers understand HTTP/2 from both a **practical** and **architectural** perspective.

Instead of only describing the protocol specification, the articles focus on how developers interact with HTTP/2 through software libraries and real implementations.

## Learning Path

The materials in this repository will gradually cover the following topics.

### Foundations

- Why HTTP/2 exists
- HTTP/1 limitations
- HTTP/2 connection model

### Core Protocol Concepts

- Frames
- Streams
- Stream state machine
- Multiplexing
- Flow control

### Programming with HTTP/2

- Event-driven APIs
- HTTP/2 library design
- Stream lifecycle in code
- Implementing HTTP/2 clients and servers

### Implementation Insights

- How HTTP/2 libraries expose protocol events
- Observing HTTP/2 traffic
- Understanding frame logs

## Status

This repository is currently under development.  
New articles will be added over time.

## License

Documentation in this repository is licensed under **Creative Commons Attribution 4.0 International (CC BY 4.0)**.
