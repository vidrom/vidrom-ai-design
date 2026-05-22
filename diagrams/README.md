# Vidrom Architecture Diagrams

This folder captures the live architecture and main runtime flows reflected by the current codebase.

- `01-system-landscape.md`: End-to-end product landscape across apps, backend, portals, and cloud services.
- `02-runtime-backend-topology.md`: Internal runtime shape of the signaling backend and its stateful subsystems.
- `03-home-app-architecture.md`: Home app composition, notification stack, and call control boundaries.
- `04-intercom-app-architecture.md`: Intercom app composition, provisioning, persistent signaling, and WebRTC hosting.
- `05-incoming-call-flow.md`: Intercom-to-resident call flow with push fanout, first-accept-wins, and media setup.
- `06-watch-flow.md`: Camera watch flow and the priority rules between watch and calls.
- `07-provisioning-and-token-flow.md`: Device provisioning, apartment resolution, and push-token registration flow.
- `08-aws-deployment-topology.md`: AWS infrastructure and deployment boundaries from CDK to runtime services.
- `09-database-entities-and-ownership.md`: Database-focused diagram of core entities, join tables, operational records, and authorization boundaries.

These diagrams are based on the current implementation in the home app, intercom app, signaling server, Lambda portal APIs, and CDK stack.