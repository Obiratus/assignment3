# Assignment 3: Auditing X’s Recommender System for Systemic Risks
**Ubiquitous Computing, HS2025 — University of St. Gallen**  
**Author:** Simon Dudler  
**Date:** 11.11.2024

---

## 1. Systemic Risk Framing

### 1.1 Overview of Systemic Risks under the DSA
- Definition of systemic risks according to the EU Digital Services Act (Article 27).
- Key risk areas:
    - Threats to democratic processes
    - Violations of fundamental rights
    - Risks to information integrity
    - Risks to public security

### 1.2 Recommender Systems and Systemic Risks
- How recommender systems influence information flows and social dynamics.
- Common mechanisms of systemic risk:
    - Amplification of harmful or misleading content
    - Polarization and echo chambers
    - Manipulation of civic discourse
    - Bias and discrimination through ranking models

### 1.3 Framing for This Audit
- How this framing guides your audit of X’s recommender system.
- Specific focus on engagement-based ranking and Grok integration.

---

## 2. Audit of Engagement-Based Ranking (RQ1)

### 2.1 Engagement Signals in X’s Recommender System
- Identified engagement signals (e.g., likes, replies, reposts, blocks, mutes, dwell time).
- Description of where these are defined in the code (with class/file references).

### 2.2 Ranking Process Overview
- How signals are processed and weighted in the ranking pipeline.
- Key modules, functions, and data flow (with references to code paths such as  
  `home-mixer/server/src/main/scala/com/twitter/home_mixer/product`).

### 2.3 Link to Systemic Risks
- How engagement-based ranking can lead to systemic risks.
- Examples or observed patterns of amplification or bias.
- Supported or inferred conclusions.

---

## 3. Audit of Grok Integration (RQ2)

### 3.1 Grok’s Role in the Recommendation Pipeline
- Evidence of Grok’s integration (code files, documentation).
- Its placement (e.g., reranking, classification, summarization).

### 3.2 Inputs, Outputs, and Functions
- Data or content Grok processes.
- Interaction with other components of the recommender system.

### 3.3 Potential Implications for Systemic Risks
- Possible effects on information integrity, fairness, or transparency.
- Risks from generative or personalized outputs.

---

## 4. Evidence and Traceability

### 4.1 Code-Level Evidence
| Finding | File Path | Class/Function | Line Numbers | Supported / Inferred |  
|----------|------------|----------------|---------------|----------------------|  
| Example: Dwell time weighting | `/server/src/main/scala/.../EngagementRanker.scala` | `EngagementRanker` | 220–245 | Supported |  

### 4.2 Documentation and External Sources
- Source links (blog posts, GitHub repos, technical explanations).
- Explanatory notes for each finding.

---

## 5. Discussion and Risk Interpretation

### 5.1 Summary of Key Findings
- Recap of findings from both RQ1 and RQ2.

### 5.2 Systemic Risk Implications under the DSA
- Which risk categories are most relevant (democratic processes, information integrity, etc.).

### 5.3 Limitations and Open Questions
- Gaps in public transparency or code accessibility.
- Potential next steps or further audit needs.

---

## 6. References
- EU DSA Article 27
- X Open Source Repository: [https://github.com/twitter/the-algorithm](https://github.com/twitter/the-algorithm)
- X Engineering Blog: [https://blog.x.com/engineering/...](https://blog.x.com/engineering/...)
- Example Audit (ACM Paper): [https://dl.acm.org/doi/pdf/10.1145/3442188.3445928](https://dl.acm.org/doi/pdf/10.1145/3442188.3445928)
- Additional cited materials

---

## Appendix (Optional)
- Extended code listings
- Screenshots or diagrams of data flow
- Supporting excerpts from external documentation  
