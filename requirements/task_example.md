**CONTEXT**
Building Brick service as GT Integration Layer for Bukuwarung's lending platform.
Migrating from LOS direct integrations → centralized integration hub in Brick.

Current: LOS directly calls VIDA, scattered integration logic
Goal: Clean boundaries, easier to maintain/scale, better observability

**TECHNICAL SCOPE**

Integrations:
1. VIDA DigiSign - borrower + lender (PoA) signatures, MIGRATE from LOS
2. BNI API - funding, disbursement, spreading, async callbacks
3. Notification Service - SMS/WA/PN via LMS Veefin
4. LMS Veefin - repayment updates from BNI callbacks

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
2. requirements/los_lms_business_architecture.xml
3. LND-4473 ticket
4. [API docs]

**OUTPUT**
RFC (3000–5000 words) following template → save as `.md` in `rfc/` folder  
Audience: Senior Engineers, Architects, Product Team  
Focus: Practical implementation, clear trade-offs, safe migration plan
