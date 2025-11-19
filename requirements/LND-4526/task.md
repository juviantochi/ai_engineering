**CONTEXT**
Building Brick service as Data Point Layer for Bukuwarung's lending platform integrating with veefin.
Centralized integration hub in Brick.

Goal: Clean boundaries, easier to maintain/scale, better observability

**TECHNICAL SCOPE**

Stack: Java, Spring Boot, Postgres  
Performance: 10K applications/day, <5s p95 latency  
Security: Audit trail for signatures/payments

**KEY DECISIONS**
1. VIDA migration: big-bang vs phased vs strangler pattern?
2. Communication patterns: sync vs async per integration?
3. Handling partial failures in multi-step workflows?
4. API design: REST/GraphQL/event-driven?

Must Include:
- Migration strategy with rollback plan
- Error handling per integration
- Sequence diagrams for critical flows
- Security & observability approach

Must Avoid:
- Monolithic design
- Tight coupling between integrations
- Synchronous chains

**REFERENCES**
1. rfc/template.md
2. LND-4526 ticket
3. PRD.md

**OUTPUT**
RFC (3000–5000 words) following template → save as `.md` in `rfc/` folder  
Audience: Senior Engineers, Architects, Product Team  
Focus: Practical implementation, clear trade-offs, safe migration plan
