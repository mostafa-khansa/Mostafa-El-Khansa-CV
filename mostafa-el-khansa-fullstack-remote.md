# MOSTAFA EL KHANSA

## Senior Full Stack Engineer

+961 81 761 183 | mostafa.khansa.lb@gmail.com
[linkedin.com/in/mostafa-el-khansa](https://www.linkedin.com/in/mostafa-el-khansa/) | [github.com/mostafa-khansa](https://github.com/mostafa-khansa)
Beirut, Lebanon (UTC+3) - open to remote, full EU overlap and US-East mornings

---

## SUMMARY

Full Stack engineer with 13+ years building and running business critical web products end to end.
I shipped the ERP platform that 2,000+ companies run their daily operations on, and the
cross platform POS that syncs to it. Strongest in backend architecture, relational data modeling,
and API design.

---

## PROFESSIONAL EXPERIENCE

### Senior Software Engineer - Wizard Solutions (full-time)

*Jal El Dib, Lebanon | Feb 2018 – Present*

- Led the migration of a legacy desktop ERP to a web platform (PHP, MySQL, JavaScript), owning
  application architecture and delivery. The platform serves **2,000+ business clients across
  Lebanon** on a recurring revenue model.
- Started as the sole application engineer with one senior database developer. As the team grew
  to **7 engineers**, I led and coached developers, and now work alongside other lead engineers.
- Rebuilt the legacy POS as a cross-platform **Flutter + SQLite** application with local-first data,
  integrated with the core ERP through a REST API I designed and maintain.
- Fixed **silent data loss** in the shared sales/orders/purchases document form: it posted
  **10,000+ individual fields**, so invoices past ~100 line items saved partially with no error.
  Replaced it with a **single JSON payload** — removing the limit rather than raising PHP's
  `max_input_vars`.
- Identified missing **commit/rollback** on multi-step document saves (e.g. invoice → inventory →
  accounting). Raised it with leadership and aligned the team on wrapping those flows in database
  transactions, and on surfacing failures instead of swallowing them.
- Traced a **2+ minute** invoice save the database developer could not isolate to a nested function
  inside a **BEFORE INSERT** trigger; save time dropped to **3 seconds or less**.
- Extended **Yii 1.1** logging so production errors include tenant, logged-in user, originating SQL,
  and a full stack trace - cutting diagnosis time on a large codebase of pages, procedures, functions,
  and triggers. Logs are viewed in **AWS CloudWatch** (pipeline owned by DevOps).

### Software Developer - Al-Kaff For Computer (remote, part-time)

*Beirut, Lebanon | Sep 2015 – Feb 2026*

- Rewrote the legacy payroll-submission web app from scratch on a current PHP version, and
  maintained other internal applications.
- Moved hosting off a Canada-based private provider onto a **Linode** Linux server I set up (Apache,
  MySQL), and led the database migration — cutting hosting from **~$4,000/month to $7.50/month**
  including backups.

### Software Developer - Al-Kaff For Computer (full-time)

*Riyadh, KSA | Mar 2013 – Sep 2015*

- As the **sole developer**, built internal web applications automating **monthly payroll processing**
  for **400 companies** (2,000+ employees each) and **secure member ATM card issuance** and
  lifecycle operations.
- Resolved critical performance bottlenecks: migrated **MyISAM → InnoDB**, redesigned core table
  structures, eliminated N+1 queries, and refactored business logic cutting payroll processing
  from **10 minutes with timeouts to 5 seconds or less**.

---

## TECHNICAL SKILLS

- **Frontend:** React, Next.js, TypeScript, JavaScript, Flutter (Dart)
- **Backend:** Node.js, PHP (Yii 1.1), Python, REST API design
- **Databases:** PostgreSQL, MySQL, SQLite, MongoDB, Prisma ORM
- **Architecture:** Modular monolith, REST, relational data modeling, performance tuning
- **Platform:** Linode, GitLab, Git, Vercel, Render, Supabase, Neon

---

## PROJECTS

> Fill this in - it is the single highest-impact thing you can do for a modern full-stack
> role. Two projects is enough. Each needs a live URL and a repo link.

### [Project name] - [live URL] | [repo]

- One line on what it does and who it is for.
- Stack: Next.js, TypeScript, PostgreSQL/Neon, Prisma, deployed on Vercel.
- One line on the interesting technical part (auth, data model, real-time, or a hard constraint).

### [Project name] - [live URL] | [repo]

- Same shape. Add RAG / LangChain later only if a project actually uses them.

---

## EDUCATION

**BSc, Computer Science** - Lebanese International University, Sep 2012
