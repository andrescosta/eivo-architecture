---
theme: default
title: The Jobico Project
author: Jobico
highlighter: shiki
transition: fade
mdc: true
layout: none
fonts:
  sans: 'Inter'
  mono: 'Fira Code'
---

<style>
/* ── Force slides to fill full height ───────────────────── */
.slidev-layout {
  padding: 0 !important;
  height: 551px !important;
}
.slidev-layout > div {
  height: 551px !important;
  box-sizing: border-box;
}

/* ── Global palette ─────────────────────────────────────── */
:root {
  --c-dark:    #0F1724;
  --c-navy:    #1A2A4A;
  --c-mid:     #1E3A5F;
  --c-steel:   #2E5B8A;
  --c-ice:     #C8DEFF;
  --c-amber:   #E8A020;
  --c-amberlt: #F5C842;
  --c-muted:   #7A92B0;
  --c-offwhite:#F4F7FC;
  --c-cardbdr: #DCE8F7;
}

.label {
  font-size: 0.62rem;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: var(--c-muted);
}

.amber { color: var(--c-amber); }
.amberlt { color: var(--c-amberlt); }
.ice { color: var(--c-ice); }
.muted { color: var(--c-muted); }
.steel { color: var(--c-steel); }

.rule-amber {
  height: 3px;
  background: var(--c-amber);
  border: none;
  margin: 0.6rem 0 1rem;
}

.card {
  background: white;
  border: 1px solid var(--c-cardbdr);
  border-left: 5px solid var(--c-amber);
  padding: 0.85rem 1.1rem;
  border-radius: 3px;
  box-shadow: 0 3px 10px rgba(0,0,0,0.08);
}

.dark-card {
  background: var(--c-mid);
  border-top: 5px solid var(--c-amber);
  padding: 1.2rem 1.4rem;
  border-radius: 3px;
  box-shadow: 0 4px 14px rgba(0,0,0,0.18);
}
.dark-card.steel-accent { border-top-color: var(--c-steel); }
.dark-card.green-accent  { border-top-color: #3A7D44; }

.badge {
  display: inline-block;
  font-size: 0.58rem;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  padding: 0.2rem 0.7rem;
  border-radius: 2px;
  margin-top: 0.6rem;
  background: rgba(232,160,32,0.18);
  color: var(--c-amber);
}
.badge.steel { background: rgba(46,91,138,0.25); color: #7ABAFF; }
.badge.green { background: rgba(58,125,68,0.25); color: #7ACC8A; }

.layer {
  padding: 0.65rem 1.1rem;
  border-radius: 3px;
  margin-bottom: 0.5rem;
  box-shadow: 0 3px 10px rgba(0,0,0,0.15);
}
.layer-exp   { background: var(--c-amber); }
.layer-sdk   { background: var(--c-steel); }
.layer-cloud { background: var(--c-mid); }
.layer-infra { background: var(--c-navy); }
.layer h3 { font-size: 0.9rem; font-weight: 700; color: #fff; margin: 0 0 0.1rem; }
.layer p  { font-size: 0.72rem; color: rgba(255,255,255,0.8); margin: 0; }

.section-label {
  position: absolute;
  top: 1.1rem;
  right: 1.8rem;
  font-size: 0.6rem;
  letter-spacing: 0.2em;
  text-transform: uppercase;
  color: var(--c-muted);
}

.track-num {
  font-size: 2.6rem;
  font-weight: 800;
  line-height: 1;
  margin-bottom: 0.3rem;
}

.dot-row {
  display: flex;
  align-items: flex-start;
  margin-bottom: 0.55rem;
  font-size: 0.82rem;
  color: var(--c-ice);
  line-height: 1.4;
}

.dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  min-width: 8px;
  border-radius: 1px;
  background: var(--c-amber);
  margin-right: 0.5rem;
  margin-top: 5px;
  flex-shrink: 0;
}
.dot.steel { background: var(--c-steel); }
</style>

<div style="background:#0F1724;height:551px;display:flex;flex-direction:column;padding:0;position:relative;overflow:hidden;">
  <div style="position:absolute;left:0;top:0;width:18px;height:551px;background:#E8A020;"></div>
  <div style="position:absolute;left:18px;top:0;width:7px;height:551px;background:#2E5B8A;"></div>
  <div style="position:absolute;right:-60px;top:-80px;font-size:26rem;font-weight:900;color:#1E3A5F;line-height:1;user-select:none;">J</div>
  <div style="padding:3rem 3rem 0 3.5rem;flex:1;display:flex;flex-direction:column;justify-content:center;">
    <div style="font-size:3.6rem;font-weight:900;letter-spacing:0.18em;color:#fff;line-height:1;">JOBICO</div>
    <div style="height:4px;width:220px;background:#E8A020;margin:1rem 0;"></div>
    <div style="font-size:1.5rem;color:#C8DEFF;margin-bottom:0.5rem;">The Jobico Project</div>
    <div style="font-size:0.8rem;letter-spacing:0.15em;color:#7A92B0;">Eivo · Agentic Development · Consulting</div>
  </div>
  <div style="padding:0 0 1.5rem 3.5rem;font-size:0.7rem;color:#7A92B0;">2025</div>
</div>

---
layout: none
---

<div style="background:#1A2A4A;height:551px;padding:2rem 2.4rem;position:relative;box-sizing:border-box;">
  <div class="section-label">Overview</div>
  <h1 style="font-size:2.4rem;font-weight:800;color:#fff;margin:0 0 0.2rem;">What is Jobico?</h1>
  <hr class="rule-amber" />
  <p style="color:#C8DEFF;font-size:0.9rem;margin-bottom:1.8rem;max-width:780px;line-height:1.65;">
    Jobico is an umbrella company that brings together an AI-powered educational platform and the expertise gained from building it — creating two distinct but connected offerings.
  </p>
  <div style="display:grid;grid-template-columns:1fr 1fr;gap:1.2rem;">
    <div class="dark-card">
      <div style="font-size:1.5rem;font-weight:800;color:#fff;margin-bottom:0.15rem;">Eivo</div>
      <div style="font-size:0.62rem;letter-spacing:0.18em;text-transform:uppercase;color:#E8A020;margin-bottom:0.8rem;">The Product</div>
      <p style="font-size:0.82rem;color:#C8DEFF;margin:0;line-height:1.6;">An AI-powered educational platform for building immersive, agentic learning experiences. Coursework, content creation, and adaptive learning — all in one.</p>
    </div>
    <div class="dark-card steel-accent">
      <div style="font-size:1.5rem;font-weight:800;color:#fff;margin-bottom:0.15rem;">Consulting</div>
      <div style="font-size:0.62rem;letter-spacing:0.18em;text-transform:uppercase;color:#7ABAFF;margin-bottom:0.8rem;">The Expertise</div>
      <p style="font-size:0.82rem;color:#C8DEFF;margin:0;line-height:1.6;">Deep knowledge in using agentic development with standard tools to build complex, real-world software. Earned by doing — not by theory.</p>
    </div>
  </div>
</div>

---
layout: none
---

<div style="background:#F4F7FC;height:551px;display:grid;grid-template-columns:2fr 3fr;box-sizing:border-box;position:relative;">
  <div class="section-label">Eivo</div>
  <div style="background:#0F1724;padding:2rem 1.6rem;display:flex;flex-direction:column;border-right:4px solid #E8A020;">
    <div style="font-size:2.4rem;font-weight:900;letter-spacing:0.18em;color:#fff;margin-bottom:0.5rem;">EIVO</div>
    <div style="font-size:0.9rem;color:#C8DEFF;margin-bottom:1.8rem;line-height:1.5;">AI-powered educational platform</div>
    <div v-for="item in [['Learning','Instructional content with interactive exercises'],['Practice','Individual, group, and competitive sessions'],['Evaluation','Writing and coding assessments with AI feedback'],['Content Creation','AI-collaborative authoring for educators']]" style="display:flex;margin-bottom:0.9rem;align-items:flex-start;">
      <div style="width:4px;min-width:4px;background:#E8A020;border-radius:1px;margin-right:0.7rem;margin-top:0.15rem;align-self:stretch;"></div>
      <div>
        <div style="font-size:0.8rem;font-weight:700;color:#F5C842;">{{ item[0] }}</div>
        <div style="font-size:0.7rem;color:#7A92B0;line-height:1.4;">{{ item[1] }}</div>
      </div>
    </div>
  </div>
  <div style="padding:2rem 1.8rem;display:flex;flex-direction:column;">
    <h2 style="font-size:1.15rem;font-weight:700;color:#0F1724;margin:0 0 1rem;">Platform Architecture</h2>
    <div class="layer layer-exp"><h3>Experiences</h3><p>Coursework · EiBooks · Vertical Apps</p></div>
    <div class="layer layer-sdk"><h3>SDK</h3><p>Facets (UI) · Core (Business Logic)</p></div>
    <div class="layer layer-cloud"><h3>Cloud API</h3><p>Editorial · Assistance · Social · Gaming · Identity</p></div>
    <div class="layer layer-infra"><h3>Infrastructure</h3><p>Kubernetes · PostgreSQL · Redis · AI Providers</p></div>
  </div>
</div>

---
layout: none
---

<div style="background:#0F1724;height:551px;padding:2rem 2.4rem;box-sizing:border-box;position:relative;">
  <div class="section-label">The Work</div>
  <h1 style="font-size:2.4rem;font-weight:800;color:#fff;margin:0 0 0.3rem;">Three Tracks. One Vision.</h1>
  <p style="font-size:0.88rem;color:#C8DEFF;max-width:750px;margin:0 0 1.6rem;line-height:1.6;">The work ahead builds Eivo into a complete product while generating the consulting expertise that becomes Jobico's second offering.</p>
  <div style="display:grid;grid-template-columns:1fr 1fr 1fr;gap:1rem;">
    <div class="dark-card" style="display:flex;flex-direction:column;">
      <div class="track-num" style="color:#E8A020;">01</div>
      <div style="font-size:1rem;font-weight:700;color:#fff;margin-bottom:0.7rem;line-height:1.3;">Architecture<br>Implementation</div>
      <p style="font-size:0.78rem;color:#C8DEFF;line-height:1.6;flex:1;">Complete the platform as designed — Facets SDK extraction, module completion, testing, and documentation — using agentic development with standard tools.</p>
      <div class="badge">Agentic Development</div>
    </div>
    <div class="dark-card steel-accent" style="display:flex;flex-direction:column;">
      <div class="track-num" style="color:#7ABAFF;">02</div>
      <div style="font-size:1rem;font-weight:700;color:#fff;margin-bottom:0.7rem;line-height:1.3;">Coursework</div>
      <p style="font-size:0.78rem;color:#C8DEFF;line-height:1.6;flex:1;">Define and build Eivo's flagship experience. Coursework is the product that proves the platform's value — built on top of a complete architecture, also using agentic development.</p>
      <div class="badge steel">Agentic Development</div>
    </div>
    <div class="dark-card green-accent" style="display:flex;flex-direction:column;">
      <div class="track-num" style="color:#7ACC8A;">03</div>
      <div style="font-size:1rem;font-weight:700;color:#fff;margin-bottom:0.7rem;line-height:1.3;">Agent<br>Infrastructure</div>
      <p style="font-size:0.78rem;color:#C8DEFF;line-height:1.6;flex:1;">Design and build the infrastructure for agentic content creation inside Eivo — manually, deliberately. This is foundational research: the learning here enables everything else.</p>
      <div class="badge green">Manual · Foundational</div>
    </div>
  </div>
</div>

---
layout: none
---

<div style="background:#F4F7FC;height:551px;padding:2rem 2.4rem;box-sizing:border-box;position:relative;">
  <div class="section-label">Tracks 01 & 02</div>
  <h1 style="font-size:2.1rem;font-weight:800;color:#0F1724;margin:0 0 0.3rem;">Agentic Development</h1>
  <p style="font-size:0.88rem;color:#7A92B0;margin:0 0 0.6rem;max-width:750px;">Building Eivo with standard AI tools — Claude, Cursor, and similar — rather than manual engineering alone. The process is the practice.</p>
  <hr class="rule-amber" />
  <div style="display:grid;grid-template-columns:1fr 1fr;gap:1.2rem;margin-top:0.2rem;">
    <div style="display:flex;flex-direction:column;gap:0.75rem;">
      <div class="card">
        <div style="font-size:0.88rem;font-weight:700;color:#0F1724;margin-bottom:0.3rem;">What gets built</div>
        <p style="font-size:0.78rem;color:#4A5568;margin:0;line-height:1.6;">The complete Eivo platform (Architecture Track) and Coursework (Experience Track) — real, production software.</p>
      </div>
      <div class="card">
        <div style="font-size:0.88rem;font-weight:700;color:#0F1724;margin-bottom:0.3rem;">How it gets built</div>
        <p style="font-size:0.78rem;color:#4A5568;margin:0;line-height:1.6;">Using agentic development throughout: AI agents drive implementation, guided by human judgment and architectural decisions.</p>
      </div>
      <div class="card">
        <div style="font-size:0.88rem;font-weight:700;color:#0F1724;margin-bottom:0.3rem;">Why it matters</div>
        <p style="font-size:0.78rem;color:#4A5568;margin:0;line-height:1.6;">Every decision, pattern, and failure becomes transferable knowledge — the raw material for Jobico's consulting practice.</p>
      </div>
    </div>
    <div style="background:#1A2A4A;padding:1.4rem;border-radius:3px;box-shadow:0 4px 14px rgba(0,0,0,0.15);border-top:4px solid #2E5B8A;">
      <div style="font-size:1.2rem;font-weight:800;color:#fff;margin-bottom:1rem;line-height:1.3;">From Practice<br>to Portfolio</div>
      <div v-for="item in ['Proven methodology for AI-driven software delivery','Patterns for working with agents at architectural scale','Real-world case study: Eivo as the reference engagement','Tooling knowledge across the agentic development stack']" class="dot-row">
        <span class="dot"></span><span>{{ item }}</span>
      </div>
    </div>
  </div>
</div>

---
layout: none
---

<div style="background:#1A2A4A;height:551px;padding:2rem 2.4rem;box-sizing:border-box;position:relative;">
  <div class="section-label">Track 03</div>
  <h1 style="font-size:2.2rem;font-weight:800;color:#fff;margin:0 0 0.15rem;">Agent Infrastructure</h1>
  <div style="font-size:1rem;color:#E8A020;font-style:italic;margin-bottom:0.8rem;">Built manually. By design.</div>
  <p style="font-size:0.85rem;color:#C8DEFF;max-width:820px;line-height:1.65;margin-bottom:1.6rem;">The agent infrastructure is Eivo-specific: the architecture and systems that enable AI agents to create educational content, collaborate with creators, and guide learners. It is built by hand because the act of building it is how the understanding is earned.</p>
  <div style="display:grid;grid-template-columns:1fr 1fr 1fr;gap:1rem;">
    <div class="dark-card green-accent">
      <div style="font-size:0.95rem;font-weight:700;color:#fff;margin-bottom:0.6rem;">Content Generation</div>
      <p style="font-size:0.78rem;color:#C8DEFF;margin:0;line-height:1.6;">Agents that create educational materials — lessons, exercises, assessments — through conversational collaboration with educators.</p>
    </div>
    <div class="dark-card steel-accent">
      <div style="font-size:0.95rem;font-weight:700;color:#fff;margin-bottom:0.6rem;">Creator Collaboration</div>
      <p style="font-size:0.78rem;color:#C8DEFF;margin:0;line-height:1.6;">Conversational AI that works alongside content creators, transforming educational intent into complete, structured learning experiences.</p>
    </div>
    <div class="dark-card">
      <div style="font-size:0.95rem;font-weight:700;color:#fff;margin-bottom:0.6rem;">Learner Guidance</div>
      <p style="font-size:0.78rem;color:#C8DEFF;margin:0;line-height:1.6;">Adaptive agents that support individual learners — analyzing progress, generating targeted practice, and providing contextual feedback.</p>
    </div>
  </div>
  <p style="font-size:0.75rem;color:#7A92B0;font-style:italic;margin-top:1.2rem;">Why manual? Because you cannot use agents to build the system that teaches you how agents work.</p>
</div>

---
layout: none
---

<div style="background:#F4F7FC;height:551px;padding:2rem 2.4rem;box-sizing:border-box;position:relative;">
  <div class="section-label">Portfolio</div>
  <h1 style="font-size:2.2rem;font-weight:800;color:#0F1724;margin:0 0 0.15rem;">The Jobico Portfolio</h1>
  <hr class="rule-amber" />
  <div style="font-size:0.95rem;color:#7A92B0;font-style:italic;margin-bottom:1.4rem;">Two assets. One company.</div>
  <div style="display:grid;grid-template-columns:1fr 1fr;gap:1.2rem;">
    <div style="background:#0F1724;padding:1.6rem;border-radius:3px;border-top:5px solid #E8A020;box-shadow:0 4px 14px rgba(0,0,0,0.2);">
      <div style="font-size:1.7rem;font-weight:800;color:#fff;margin-bottom:0.1rem;">Eivo</div>
      <div style="font-size:0.62rem;letter-spacing:0.18em;text-transform:uppercase;color:#E8A020;margin-bottom:1rem;">Product</div>
      <div v-for="item in ['AI-powered educational platform','Coursework as the flagship experience','Agent infrastructure for content creation','Multi-provider LLM integration','Cloud-native, extensible architecture']" class="dot-row">
        <span class="dot"></span><span>{{ item }}</span>
      </div>
    </div>
    <div style="background:#1E3A5F;padding:1.6rem;border-radius:3px;border-top:5px solid #2E5B8A;box-shadow:0 4px 14px rgba(0,0,0,0.18);">
      <div style="font-size:1.7rem;font-weight:800;color:#fff;margin-bottom:0.1rem;">Consulting</div>
      <div style="font-size:0.62rem;letter-spacing:0.18em;text-transform:uppercase;color:#F5C842;margin-bottom:1rem;">Expertise</div>
      <div v-for="item in ['Agentic development methodology','Building complex software with AI tools','Patterns for agent-driven architecture','Real engagement: Eivo as the case study','Practical, earned — not theoretical']" class="dot-row">
        <span class="dot steel"></span><span>{{ item }}</span>
      </div>
    </div>
  </div>
</div>

---
layout: none
---

<div style="background:#0F1724;height:551px;display:flex;flex-direction:column;align-items:center;justify-content:center;position:relative;overflow:hidden;text-align:center;">
  <div style="position:absolute;top:0;left:0;right:0;height:18px;background:#E8A020;"></div>
  <div style="position:absolute;bottom:0;left:0;right:0;height:18px;background:#2E5B8A;"></div>
  <div style="position:absolute;right:-80px;top:-60px;font-size:26rem;font-weight:900;color:#1E3A5F;line-height:1;user-select:none;">J</div>
  <div style="position:relative;z-index:1;">
    <div style="font-size:3.8rem;font-weight:900;letter-spacing:0.2em;color:#fff;margin-bottom:1.2rem;">JOBICO</div>
    <div style="font-size:1.25rem;color:#C8DEFF;line-height:1.7;margin-bottom:1.4rem;">Building the future of learning.<br>Through AI. By doing.</div>
    <div style="width:160px;height:3px;background:#E8A020;margin:0 auto 1.2rem;"></div>
    <div style="font-size:0.75rem;letter-spacing:0.18em;color:#7A92B0;">Eivo · Agentic Development · Consulting</div>
  </div>
</div>
