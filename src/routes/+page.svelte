<script lang="ts">
  import { onMount, onDestroy } from "svelte";

  // ----------------------------
  // Types
  // ----------------------------

  type Question = {
    domain: string;
    lesson: string;
    question: string;
    correctAnswer: string;
    dummyAnswers?: string[];
    __index?: number; // original index in allQuestions
  };

  type Option = {
    text: string;
    correct: boolean;
  };

  // ----------------------------
  // Top-level state
  // ----------------------------

  let loading = true;
  let error: string | null = null;

  let allQuestions: Question[] = [];
  let domains: string[] = [];

  // Session config
  let selectedDomains = new Set<string>();
  let minQuestions = 10;

  // Active session
  let sessionQuestions: Question[] = [];
  let sessionStarted = false;
  let currentIndex = 0;
  let currentOptions: Option[] = [];

  let status: "idle" | "correct" | "incorrect" = "idle";
  let statusMessage = "";

  // Timer ID for auto-advancing after a correct answer.
  // We keep track of it so we can cancel if needed.
  let autoAdvanceTimeout: number | null = null;

  // ----------------------------
  // Lifecycle: load questions
  // ----------------------------

  onMount(async () => {
    try {
      const res = await fetch("/Quiz.json");
      if (!res.ok) {
        throw new Error(`Failed to load Quiz.json: ${res.status}`);
      }

      const data: Question[] = await res.json();

      // Attach original indices
      allQuestions = data.map((q, idx) => ({ ...q, __index: idx }));

      // Collect unique domains
      const domainSet = new Set<string>();
      allQuestions.forEach((q) => {
        domainSet.add(q.domain || "General");
      });
      domains = Array.from(domainSet).sort();

      // Default: select all domains
      selectedDomains = new Set(domains);
    } catch (err: any) {
      console.error(err);
      error = err?.message ?? "Failed to load questions.";
    } finally {
      loading = false;
    }
  });

  // Make sure timers are cleaned up if the component goes away.
  onDestroy(() => {
    if (autoAdvanceTimeout !== null) {
      clearTimeout(autoAdvanceTimeout);
      autoAdvanceTimeout = null;
    }
  });

  // ----------------------------
  // Helpers
  // ----------------------------

  function toggleDomain(domain: string) {
    if (selectedDomains.has(domain)) {
      selectedDomains.delete(domain);
    } else {
      selectedDomains.add(domain);
    }
    // Reassign to trigger Svelte reactivity with Set
    selectedDomains = new Set(selectedDomains);
  }

  function shuffleArray<T>(arr: T[]): T[] {
    const a = [...arr];
    for (let i = a.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [a[i], a[j]] = [a[j], a[i]];
    }
    return a;
  }

  // Build a pool of distractors from OTHER questions' correct answers.
  function getGlobalDistractorPool(excludeIndex: number | undefined): string[] {
    if (excludeIndex === undefined) return [];
    const pool: string[] = [];

    allQuestions.forEach((q, idx) => {
      if (idx === excludeIndex) return;
      if (!q || typeof q.correctAnswer !== "string") return;
      pool.push(q.correctAnswer);
    });

    return pool;
  }

  // Build the answer options for a single question:
  // - Always include the correct answer
  // - Include per-question dummy answers if available
  // - Top up with correct answers from other questions (global pool)
  function buildOptionsForQuestion(q: Question): Option[] {
    const correctText = q.correctAnswer;
    const options: Option[] = [];
    const seen = new Set<string>();

    // Always include correct answer
    options.push({ text: correctText, correct: true });
    seen.add(correctText);

    // Per-question dummy answers (if any)
    const perQ = Array.isArray(q.dummyAnswers) ? [...q.dummyAnswers] : [];
    perQ.forEach((t) => {
      if (typeof t === "string" && !seen.has(t)) {
        options.push({ text: t, correct: false });
        seen.add(t);
      }
    });

    // Top up from other questions' correct answers
    const desiredOptionCount = 4;
    const needed = Math.max(0, desiredOptionCount - options.length);

    if (needed > 0) {
      const pool = shuffleArray(
        getGlobalDistractorPool(q.__index).filter((t) => !seen.has(t))
      );
      for (let i = 0; i < needed && i < pool.length; i++) {
        const text = pool[i];
        options.push({ text, correct: false });
        seen.add(text);
      }
    }

    return shuffleArray(options);
  }

  // Prepare the UI for the current question (options + reset status).
  function prepareCurrentQuestion() {
    const q = sessionQuestions[currentIndex];
    currentOptions = buildOptionsForQuestion(q);
    status = "idle";
    statusMessage = "";
  }

  // Helper: clear any existing auto-advance timer
  function clearAutoAdvanceTimer() {
    if (autoAdvanceTimeout !== null) {
      clearTimeout(autoAdvanceTimeout);
      autoAdvanceTimeout = null;
    }
  }

  // Helper: schedule automatic move to the next question
  // after a short delay (e.g. 2 seconds) when the answer is correct.
  function scheduleAutoAdvance() {
    clearAutoAdvanceTimer();

    // If this is the last question, we still call nextQuestion()
    // which will gracefully end the session.
    autoAdvanceTimeout = window.setTimeout(() => {
      nextQuestion(true); // `true` = auto-advance
    }, 1000); // "couple of seconds"
  }

  // Start (or restart) a quiz session.
  function startSession() {
    if (!allQuestions.length) return;

    // If we had a pending auto-advance timer from a previous session, clear it.
    clearAutoAdvanceTimer();

    // Determine which domains to use
    let activeDomains = Array.from(selectedDomains);
    if (!activeDomains.length) {
      // If user unchecks everything, fall back to all
      activeDomains = domains;
      selectedDomains = new Set(domains);
    }

    // Filter questions by chosen domains
    let candidates = allQuestions.filter((q) =>
      activeDomains.includes(q.domain || "General")
    );

    if (!candidates.length) {
      error = "No questions found for the selected domains.";
      return;
    }

    // Shuffle candidates
    candidates = shuffleArray(candidates);

    // Trim or keep all depending on minQuestions
    if (candidates.length > minQuestions) {
      candidates = candidates.slice(0, minQuestions);
    }

    sessionQuestions = candidates;
    currentIndex = 0;
    sessionStarted = true;
    prepareCurrentQuestion();
  }

  // Handle an answer click:
  // - If correct: show success, schedule auto-advance
  // - If incorrect: show error AND re-queue the same question at the end
  function handleAnswerClick(option: Option) {
    if (status !== "idle") return;

    // Clear any pending auto-advance just in case
    clearAutoAdvanceTimer();

    if (option.correct) {
      status = "correct";
      statusMessage = "Correct!";

      // Automatically advance to the next question after a short delay
      scheduleAutoAdvance();
    } else {
      status = "incorrect";
      statusMessage = "Incorrect.";

      // Re-queue logic for wrong answers:
      // We take the current question and push a reference to the end
      // of `sessionQuestions`, so the user will see it again later.
      const currentQuestionRef = sessionQuestions[currentIndex];

      // To avoid adding multiple copies *ahead* of us,
      // we only append if there isn't already another copy
      // later in the list.
      const remaining = sessionQuestions.slice(currentIndex + 1);
      const alreadyQueued = remaining.includes(currentQuestionRef);

      if (!alreadyQueued) {
        sessionQuestions = [...sessionQuestions, currentQuestionRef];
      }

      // NOTE: we do NOT auto-advance on incorrect answers.
      // The user must click "Next Question", but the question
      // will come back later in the queue.
    }
  }

  // Move to the next question in the session.
  // `auto` is just a flag so we know if it was called from the timer,
  // but we don't actually need different behavior right now.
  function nextQuestion(auto = false) {
    if (!sessionStarted) return;

    // Any time we manually move, kill any pending auto-advance timer.
    clearAutoAdvanceTimer();

    if (currentIndex < sessionQuestions.length - 1) {
      currentIndex += 1;
      prepareCurrentQuestion();
    } else {
      // End of session
      status = "idle";
      statusMessage = "";
      sessionStarted = false;
    }
  }

  // Reactive statement: keep a handy reference
  // to the "current" question for the template.
  $: currentQuestion = sessionStarted ? sessionQuestions[currentIndex] : null;
</script>

{#if loading}
  <div class="text-center py-5">
    <div class="spinner-border text-primary" role="status">
      <span class="visually-hidden">Loading...</span>
    </div>
    <p class="mt-3">Loading questions…</p>
  </div>
{:else if error}
  <div class="alert alert-danger" role="alert">
    {error}
  </div>
{:else}
  <div class="row">
    <div class="col-lg-4 mb-4">
      <div class="card">
        <div class="card-header">sk-sb-quiz</div>
        <div class="card-body">
          <h5 class="card-title">Session setup</h5>
          <p class="card-text small text-muted">
            Choose one or more domains, set a minimum number of questions, then
            start a quiz session.
          </p>

          <div class="mb-3">
            <!-- svelte-ignore a11y_label_has_associated_control -->
            <label class="form-label fw-semibold">Domains</label>

            <div class="mb-3">
              <label class="form-label" for="minQuestions">
                Minimum questions for this session
              </label>
              <input
                id="minQuestions"
                type="number"
                class="form-control"
                bind:value={minQuestions}
                min="5"
                max="100"
              />
              <div class="form-text">
                We'll take up to this many questions from your chosen domains.
              </div>
            </div>

            <button class="btn btn-primary w-100" on:click={startSession}>
              {sessionStarted ? "Restart Session" : "Start Session"}
            </button>

            <div class="d-flex flex-column gap-1">
              {#each domains as domain}
                <div class="form-check">
                  <input
                    class="form-check-input"
                    type="checkbox"
                    id={"dom-" + domain}
                    checked={selectedDomains.has(domain)}
                    on:change={() => toggleDomain(domain)}
                  />
                  <label class="form-check-label" for={"dom-" + domain}>
                    {domain}
                  </label>
                </div>
              {/each}
            </div>
          </div>

          {#if sessionStarted}
            <p class="mt-3 small text-muted">
              Question {currentIndex + 1} of {sessionQuestions.length}
            </p>
          {/if}
        </div>
      </div>
    </div>

    <div class="col-lg-8">
      {#if sessionStarted && currentQuestion}
        <div class="card">
          <div
            class="card-header d-flex justify-content-between align-items-center"
          >
            <span>{currentQuestion.domain}</span>
            <span class="badge bg-secondary">
              Q{currentIndex + 1} / {sessionQuestions.length}
            </span>
          </div>
          <div class="card-body">
            {#if currentQuestion.lesson}
              <p class="d-none text-muted small mb-2">
                {currentQuestion.lesson}
              </p>
            {/if}

            <h5 class="card-title mb-3">
              {currentQuestion.question}
            </h5>

            <div class="d-grid gap-2 mb-3">
              {#each currentOptions as opt, i}
                <button
                  type="button"
                  class="btn btn-outline-primary text-start"
                  class:btn-success={status === "correct" && opt.correct}
                  class:btn-danger={status === "incorrect" && opt.correct}
                  on:click={() => handleAnswerClick(opt)}
                  disabled={status !== "idle"}
                >
                  <strong>{String.fromCharCode(65 + i)}.</strong>
                  {opt.text}
                </button>
              {/each}
            </div>

            {#if status === "correct"}
              <div class="alert alert-success py-2" role="alert">
                {statusMessage}
              </div>
            {:else if status === "incorrect"}
              <div class="alert alert-danger py-2" role="alert">
                {statusMessage}
              </div>
            {/if}

            <div class="d-flex justify-content-end mt-3">
              <button
                type="button"
                class="btn btn-secondary"
                on:click={() => nextQuestion(false)}
              >
                {currentIndex < sessionQuestions.length - 1
                  ? "Next Question"
                  : "End Session"}
              </button>
            </div>
          </div>
        </div>
      {:else}
        <div class="alert alert-info" role="alert">
          Start a session on the left to begin practicing your stack domains.
        </div>
      {/if}
    </div>
  </div>
{/if}
