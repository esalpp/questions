# questions

## .NET Knowledge Quiz

`dotnet-quiz.html` is a self-contained quiz with 58 questions covering C#, LINQ, async/await, threading and locks (including Mutex), the runtime, ASP.NET Core, EF Core and SQL.

- 46 multiple-choice questions
- 12 "write the code" questions (SQL queries, a single-instance app with a Mutex, a thread-safe singleton, coordinating two threads, LINQ, throttled async downloads, middleware and more)

**Play online:** https://esalpp.github.io/questions/

**Play offline:** download or clone the repo and double-click `dotnet-quiz.html` to open it in any browser. No install or internet connection is needed.

### How it works

- **Check one question at a time:** every question has a **Show answer** button that reveals the answer and explanation for just that question.
- **Write code questions:** type your solution, press **Show model answer**, compare, then mark yourself **I got it** or **Not quite**.
- **Reveal all answers** unlocks once every question is done, or use "Reveal all anyway". You then get your score and a per-topic breakdown.
- Explanations teach the concept behind each answer, with code examples and a "Remember" tip.
- Use the topic buttons at the top to practice one area at a time (or only the write-code questions).
- Your answers are saved in the browser, so refreshing won't lose progress. **Retake quiz** clears everything and shuffles the questions and options.

To add or edit questions, change the `QUESTIONS` array near the top of the `<script>` block. The comment above it describes the format.
