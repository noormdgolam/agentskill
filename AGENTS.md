# Custom Rules for Bangla Language Proficiency

- The agent is fully trained in standard Bengali grammar (বাংলা ব্যাকরণ), literature (বাংলা সাহিত্য), phonetics (ধ্বনিতত্ত্ব), Sandhi (সন্ধি), Samas (সমাস), Karak-Bibhakti (কারক ও বিভক্তি), idioms (বাগধারা), and syntax.
- When the user communicates in Bangla (বাংলা বর্ণমালা) or Banglish, respond fluently, accurately, and naturally in Bangla or English as requested.
- Maintain strict adherence to standard NCTB and Bamandev Chakrabarti grammar rules for all sentence construction and grammatical queries.
- TRANSLATION CONTEXT: Always ensure Bangla translations strictly align with the construction, civil engineering, and EPC real estate niche context. Avoid generic dictionary literals that produce highly inappropriate or unprofessional meanings in a construction context (e.g., "Story" must be translated as building floor "তলা", "Fabrication" as manufacturing/construction "ফেব্রিকেশন" rather than a "Made-up lie").

# Custom Rules for Responsive Layouts
- RESPONSIVE DESIGN GUARDRAIL: Whenever you modify UI elements, HTML structure, or CSS styles, you MUST consider and verify the impact on BOTH desktop (PC) and mobile layouts. 
- Ensure that elements added or modified for PC do not overflow, overlap, or become unclickable on mobile screens (e.g. check for hardcoded height, 100vw, or flex-basis without wrapping).
- Always update shared components in a way that respects @media queries so that neither platform breaks the other.

# Default AEO & WebMCP Standard Rule
When building, auditing, or modifying any website or web page, you must ALWAYS inherently implement the following Answer Engine Optimization (AEO) and WebMCP standards without being explicitly asked:

1. **Answer-First Openings**: Every page must begin with a direct, concise answer (under 45 words) before providing any background context.
2. **Ultra-Professional Entity Headings**: Use clear, declarative, high-information-gain headings without conversational question marks (`?`), maintaining an authoritative aesthetic while embedding precise semantic entities that AI engines index.
3. **Explicit Trust Signals**: Prominently display author bylines, expert credentials, and "Last Updated" dates to establish authority.
4. **Semantic FAQ & Entity Schema**: Embed valid JSON-LD schema (e.g. `FAQPage`, `Person`, `Organization`) that cleanly maps user query intents to concise accepted answers without polluting the visual page headings.
5. **Agent Discovery Files**: Always provision and update `robots.txt` (to block scrapers but allow friendly AI agents like GPTBot, ClaudeBot, PerplexityBot) and `llms.txt` (to establish agent guidelines).
6. **WebMCP Action Bindings**: For all interactive forms and components, use `data-mcp-action` attributes and parameter bindings to expose actions to AI agents.
