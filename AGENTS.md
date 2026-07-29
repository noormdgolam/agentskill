# Custom Rules for Bangla Language Proficiency

- The agent is fully trained in standard Bengali grammar (বাংলা ব্যাকরণ), literature (বাংলা সাহিত্য), phonetics (ধ্বনিতত্ত্ব), Sandhi (সন্ধি), Samas (সমাস), Karak-Bibhakti (কারক ও বিভক্তি), idioms (বাগধারা), and syntax.
- When the user communicates in Bangla (বাংলা বর্ণমালা) or Banglish, respond fluently, accurately, and naturally in Bangla or English as requested.
- Maintain strict adherence to standard NCTB and Bamandev Chakrabarti grammar rules for all sentence construction and grammatical queries.
- TRANSLATION CONTEXT: Always ensure Bangla translations strictly align with the construction, civil engineering, and EPC real estate niche context. Avoid generic dictionary literals that produce highly inappropriate or unprofessional meanings in a construction context (e.g., "Story" must be translated as building floor "তলা", "Fabrication" as manufacturing/construction "ফেব্রিকেশন" rather than a "Made-up lie").

# Custom Rules for Responsive Layouts
- RESPONSIVE DESIGN GUARDRAIL: Whenever you modify UI elements, HTML structure, or CSS styles, you MUST consider and verify the impact on BOTH desktop (PC) and mobile layouts. 
- Ensure that elements added or modified for PC do not overflow, overlap, or become unclickable on mobile screens (e.g. check for hardcoded height, 100vw, or flex-basis without wrapping).
- Always update shared components in a way that respects @media queries so that neither platform breaks the other.
