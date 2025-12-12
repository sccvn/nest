# NestJS Core Repository Overview

## Project Purpose
NestJS is a progressive Node.js framework for building efficient, scalable server-side applications. It uses modern JavaScript and is built with TypeScript (while preserving compatibility with pure JavaScript).

## Key Features
- Combines elements of OOP (Object Oriented Programming), FP (Functional Programming), and FRP (Functional Reactive Programming)
- Under the hood uses Express by default, but also provides compatibility with Fastify
- Architecture heavily inspired by Angular
- Provides an application architecture out of the box for testable, scalable, loosely coupled and maintainable applications

## Repository Type
This is the **core monorepo** for NestJS framework development, containing all core packages published to npm under the `@nestjs/*` scope.

## Current Version
- Lerna version: 11.1.9
- Node.js requirement: >= 20

## Main Dependencies
- **Core Runtime**: rxjs, reflect-metadata, iterare, tslib
- **HTTP**: express (v5), fastify, cors
- **Validation**: class-transformer, class-validator
- **WebSockets**: socket.io
- **Others**: path-to-regexp, uuid, uid

## License
MIT License

## Author
Kamil Mysliwiec

## Links
- Documentation: https://docs.nestjs.com
- Homepage: https://nestjs.com
- Repository: https://github.com/nestjs/nest
