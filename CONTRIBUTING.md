# Contributing

Thanks for helping improve Data Engineer Prep. This repo is meant to stay practical, curated, and interview-focused. Small fixes are welcome, and high-signal additions are even better.

## Good Contributions

- Typo fixes, broken-link fixes, and clearer explanations.
- New quiz questions with concise answers.
- Stronger examples for existing PySpark, SQL, data modeling, or AI-for-data-engineering topics.
- Zephyr Coffee Co. scenario ideas that make a concept feel realistic.
- Company interview patterns based on public sources or your own experience.
- External resources with a short reason for why they are better than what is already linked.

## Please Avoid

- Random link dumps.
- Self-promotional content.
- AI-generated filler that has not been checked by a human.
- Exact live interview questions copied verbatim.
- Proprietary datasets, take-home assignments, internal rubrics, interviewer names, or confidential offer details.

The curation is the point. If a link, question, or explanation does not make the reader more prepared, leave it out.

## Content Standards

Before opening a pull request, check that your change:

- Is directly useful for data engineering interview prep.
- Matches the existing tone: practical, direct, and no fluff.
- Explains the "why", not only the "what".
- Uses correct Markdown links and relative paths.
- Avoids unsupported claims. If a fact depends on a source, cite it.
- Keeps examples small enough to understand, but realistic enough to matter.

## Adding Company Interview Content

Company interview pages are useful only when they are trustworthy. Please follow these rules carefully.

For community-sourced research:

1. Copy `company_interviews/_template_online.md`.
2. Use public sources only, such as GeeksForGeeks, Glassdoor, Reddit, Medium, YouTube, Naukri, or candidate blogs.
3. Add a `Sources` section with actual URLs.
4. Summarize patterns in your own words. Do not copy-paste from other sites.
5. Prefer question patterns over exact wording.

For personal interview experiences:

1. Copy `company_interviews/_template_personal.md`.
2. Share round structure, topic areas, difficulty, prep strategy, and what the interview was testing.
3. Do not include confidential or exact question text. Rephrase as a pattern.
4. Include a "Delta from online.md" section if an `online.md` file exists for that company.

When in doubt, describe the type of question instead of the question itself.

## Adding PySpark Notebooks

New notebooks should:

- Be self-contained or clearly explain how to load any sample data.
- Start with the business problem before the code.
- Use the Zephyr Coffee Co. framing where it helps.
- Include a walkthrough, at least one exercise, and a solution section.
- Keep outputs clean before committing.
- Include a Colab badge if the notebook is meant to run in Colab.

If you add a notebook, update the relevant module README and the root `README.md` if needed.

## Adding Quizzes

Quizzes should use the existing collapsible Q&A style:

- Start with basics.
- Move into intermediate judgment.
- End with senior/interview-style tradeoffs.
- Keep answers short enough to review the night before an interview.

## Pull Request Checklist

Before opening a PR:

- Run a spell check or at least read the changed file end to end.
- Click any new links you added.
- Check that relative links work from the file's folder.
- Make sure no private or confidential information is included.
- Keep the PR focused. One topic per PR is easiest to review.

## Review Philosophy

This repo values clear, battle-tested explanations over volume. A short correction that prevents someone from learning the wrong thing is a great contribution.
