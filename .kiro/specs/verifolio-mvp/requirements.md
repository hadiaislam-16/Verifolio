# Requirements Document

## Introduction

Verifolio is an AI-powered evidence-based technical portfolio platform for students and early-career developers. Instead of relying on self-reported skills from a CV, Verifolio analyzes a candidate's uploaded CV and publicly accessible GitHub repositories to verify which skills have genuine code evidence behind them.

The platform extracts a structured, editable profile from an uploaded CV, analyzes pasted public repository URLs to discover and rate skills, identifies CV skills that lack supporting evidence in the candidate's projects, and matches the evidence-backed profile against a pasted job description. Candidates receive a shareable public profile URL they can include in job applications.

The MVP targets the candidate experience exclusively. GitHub OAuth, private repositories, recruiter accounts, and marketplace features are out of scope. Data is stored per authenticated session; HTTPS is assumed for all communication.

---

## Glossary

- **Candidate**: A student or early-career developer who creates and manages a Verifolio profile.
- **CV**: A curriculum vitae or résumé document uploaded by the Candidate in PDF or plain-text format.
- **Profile**: The structured, editable representation of a Candidate's identity, experience, education, and skills, initially populated by CV extraction.
- **Repository**: A publicly accessible GitHub repository URL provided by the Candidate.
- **Skill**: A named technical capability (e.g., "React", "REST API Design", "SQL") associated with a Candidate.
- **Declared Skill**: A Skill listed on the Candidate's CV or manually added by the Candidate to their Profile.
- **Discovered Skill**: A Skill identified by the AI Analyzer from Repository analysis that was not present as a Declared Skill.
- **Weak Claim**: A Declared Skill for which the AI Analyzer found no supporting Evidence Items in any analyzed Repository.
- **Evidence Item**: A specific artifact (repository name, file path, and a one-sentence description of the observed code pattern) that supports the presence of a Skill in a Repository.
- **Evidence Report**: A structured view associating a Skill with its Evidence Items across one or more Repositories.
- **Skill Strength**: A qualitative rating (Strong, Partial, or Inferred) assigned by the AI Analyzer to represent the depth of evidence for a Skill.
- **Job Description (JD)**: A plain-text block of text describing a role's required and preferred skills, provided by the Candidate.
- **Match Result**: The output of comparing a JD against a Candidate's evidence-backed Profile, categorizing each required skill as a Strong Match, Partial Match, or Gap.
- **Public Profile**: A read-only shareable web page presenting a Candidate's verified skills and evidence summary, accessible via a unique URL.
- **CV Extractor**: The AI component responsible for parsing a CV and producing structured Profile data.
- **AI Analyzer**: The AI component responsible for analyzing Repository content and producing Skills, Evidence Items, Skill Strength ratings, and Evidence Reports.
- **Job Matcher**: The AI component responsible for comparing a JD's required skills to a Candidate's evidence-backed Profile and producing Match Results.
- **System**: The Verifolio platform as a whole, including all frontend, backend, and AI components.

---

## Requirements

### Requirement 1: Candidate Account Management

**User Story:** As a Candidate, I want to create and manage an account, so that my profile and analysis results are securely stored and accessible across sessions.

#### Acceptance Criteria

1. THE System SHALL allow a Candidate to register using an email address and a password of at least 8 characters.
2. IF a Candidate submits a registration form with an email address already associated with an existing account, THEN THE System SHALL display an error message stating the email address is already in use.
3. WHEN a Candidate submits valid login credentials, THE System SHALL authenticate the Candidate and present the Candidate's Profile dashboard.
4. IF a Candidate submits invalid login credentials, THEN THE System SHALL display an error message and SHALL NOT grant access to any account.
5. WHEN a Candidate logs out, THE System SHALL end the authenticated session and redirect the Candidate to the login page.

---

### Requirement 2: CV Upload and AI-Powered Profile Extraction

**User Story:** As a Candidate, I want to upload my CV so that the system automatically creates a structured profile from my existing experience and skills.

#### Acceptance Criteria

1. THE System SHALL accept CV uploads in PDF and plain-text (.txt) formats.
2. IF a Candidate uploads a file in an unsupported format, THEN THE System SHALL display an error message stating the accepted file types and SHALL NOT process the file.
3. WHEN a Candidate uploads a valid CV file, THE CV Extractor SHALL extract the Candidate's name, education history, work experience, and Declared Skills and populate the Candidate's Profile with the extracted data.
4. IF the CV Extractor cannot parse a section of the CV, THEN THE System SHALL display a warning indicating which sections could not be extracted and SHALL leave those fields empty for manual completion.
5. THE System SHALL allow a Candidate to manually edit any field in the Profile at any time.
6. WHEN a Candidate uploads a new CV, THE System SHALL replace the existing Profile with the newly extracted data.

---

### Requirement 3: Repository Analysis

**User Story:** As a Candidate, I want to paste public GitHub repository URLs so that the AI can analyze my actual code and identify the skills I've demonstrated.

#### Acceptance Criteria

1. THE System SHALL allow a Candidate to add between 1 and 5 public GitHub repository URLs to their Profile.
2. IF a Candidate submits a Repository URL that is inaccessible or returns an error, THEN THE System SHALL display an error message identifying the problematic URL and SHALL NOT proceed with analysis of that Repository.
3. WHEN a Candidate initiates analysis of a Repository, THE AI Analyzer SHALL analyze the Repository's source code and identify Skills with associated Evidence Items and Skill Strength ratings.
4. WHEN the AI Analyzer completes analysis of a Repository, THE System SHALL display the identified Skills and their Evidence Items to the Candidate.
5. THE AI Analyzer SHALL identify Skills across at minimum the following dimensions: programming languages, frameworks and libraries, architectural patterns, testing practices, and tooling.

---

### Requirement 4: Skill Discovery and Weak Claim Detection

**User Story:** As a Candidate, I want the system to discover skills I haven't listed on my CV but have demonstrated in my code, and to flag CV skills I've claimed but can't back up with evidence, so that my profile is both complete and honest.

#### Acceptance Criteria

1. WHEN the AI Analyzer identifies a Skill in a Repository that is not present in the Candidate's Declared Skills, THE System SHALL add that Skill to the Candidate's Profile as a Discovered Skill with a visual indicator distinguishing it from Declared Skills.
2. WHEN a Candidate accepts a Discovered Skill, THE System SHALL add the Skill to the Candidate's confirmed skill set and retain its associated Evidence Items.
3. WHEN a Candidate dismisses a Discovered Skill, THE System SHALL remove the Skill from the Candidate's Profile.
4. WHEN Repository analysis completes, THE System SHALL identify any Declared Skill for which no Evidence Items were found across all analyzed Repositories and SHALL display those skills as Weak Claims with a distinct visual indicator.
5. THE AI Analyzer SHALL produce at least one Evidence Item for each Discovered Skill, identifying the Repository name, file path, and a one-sentence description of the observed code pattern.

---

### Requirement 5: Evidence Reports

**User Story:** As a Candidate, I want to see evidence reports for each of my skills, so that I can understand what proof backs each skill and communicate it confidently.

#### Acceptance Criteria

1. THE System SHALL generate an Evidence Report for each Skill associated with the Candidate's Profile that has at least one Evidence Item.
2. WHEN a Candidate views an Evidence Report, THE System SHALL display the Skill name, Skill Strength rating, and for each Evidence Item: the Repository name, file path, and a one-sentence description of the observed code pattern.
3. THE AI Analyzer SHALL assign a Skill Strength of "Strong" when three or more substantive Evidence Items are found across analyzed Repositories, "Partial" when one or two Evidence Items are found, and "Inferred" when the Skill is implied by project context but no direct code artifacts were identified.
4. WHEN a new Repository analysis completes, THE System SHALL update existing Evidence Reports to incorporate new Evidence Items from that Repository.

---

### Requirement 6: Job Description Matching

**User Story:** As a Candidate, I want to paste a job description and see how my evidence-backed skills compare to the job's requirements, so that I can assess my fit and identify gaps.

#### Acceptance Criteria

1. THE System SHALL provide a text input field for a Candidate to paste a Job Description.
2. WHEN a Candidate submits a Job Description, THE Job Matcher SHALL extract the required and preferred skills from the Job Description text and compare them against the Candidate's evidence-backed Profile.
3. WHEN the Job Matcher produces Match Results, THE System SHALL display each required skill categorized as a Strong Match, Partial Match, or Gap, along with a summary count of each category.
4. WHEN a Candidate selects a Strong Match or Partial Match skill in the Match Results, THE System SHALL display the associated Evidence Report for that Skill.
5. WHEN a Candidate selects a Gap skill in the Match Results, THE System SHALL display the skill name and a description of what type of evidence would close the gap.
6. THE System SHALL allow a Candidate to run multiple Job Description matches against the same Profile without re-triggering Repository analysis.

---

### Requirement 7: Public Shareable Profile

**User Story:** As a Candidate, I want a shareable public URL for my Verifolio profile, so that I can share my evidence-backed skills with recruiters or include it in job applications.

#### Acceptance Criteria

1. THE System SHALL assign each Candidate a unique Public Profile URL upon account creation.
2. THE Public Profile SHALL be accessible to any visitor with the URL without requiring authentication.
3. THE Public Profile SHALL display the Candidate's name, confirmed skills grouped by Skill Strength, and the Evidence Report summary for each skill.
4. THE Public Profile SHALL NOT display the Candidate's contact information, education history, or work experience.
5. THE System SHALL allow a Candidate to toggle the visibility of individual skills on the Public Profile without removing them from the Candidate's internal Profile.
6. WHEN a Candidate updates their Profile or Evidence Reports, THE System SHALL reflect the updates on the Public Profile upon the next page load.
