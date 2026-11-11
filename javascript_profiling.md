Ask the user for a web page URL. Profile the JavaScript to find quick, non-obvious performance wins.
Skip the obvious advice (bundle size, image formats, caching headers). Look inside V8 for code
patterns that cost more than they should.

Before you start
- Ask the engineer once, upfront: "Can I clone the V8 and Chromium sources (read-only, for
  root-causing) into a local OSS directory I can access, such as ~/oss? Which directory should I
  use?" Don't clone anything without a yes.
- If they agree, match the source to the browser you profile (V8 and Chrome versions from
  chrome://version, or node -p process.versions.v8 for Node):
  - V8: git clone --filter=blob:none https://chromium.googlesource.com/v8/v8.git, then check
    out the matching tag. This one is small.
  - Chromium (Blink lives in third_party/blink): git clone --filter=blob:none
    https://chromium.googlesource.com/chromium/src.git, then check out the matching tag. Don't
    run gclient sync, because reading the source doesn't need the dependencies. It's still tens
    of GB, so say so when you ask.
  - Reuse existing checkouts and record their paths and versions so later sessions can find them.
- If they decline, continue without the sources and mark any explanation of engine behaviour
  as unconfirmed.

Setup
- Use a cold load with CPU throttling (about 4–8x) and record a trace with the
  disabled-by-default-v8.runtime_stats category (Runtime Call Stats).
- If you're driving Chrome with Puppeteer or Playwright, call
  Runtime.setMaxCallStackSizeToCapture with size 0. Otherwise V8 captures deep stacks and
  reparses big functions, which inflates parse and compile time.
- Load the page 2–3 times and treat only counters that hold steady across loads as findings.

Read the RCS table, sorted by time and by call count. Things that point to a fixable pattern:
   - lazy compile or PreParse taking a large share: huge functions compiled late, or code
     parsed twice (wrap the immediately-called ones in parentheses, split the giants)
   - many StoreIC_Miss / KeyedStoreIC_Miss / LoadIC_Miss: polymorphic or megamorphic property
     access, objects built key by key, x = undefined then reassigned
   - PrototypeMap, map transitions, UpdateProtector: shapes churning, or code writing to
     built-in prototypes late (polyfills) and invalidating optimizations
   - NewClosure: closures allocated in hot paths or at module load
   - SetKeyedProperty, or objects falling into dictionary mode: objects with many dynamic
     keys, spread copies of large objects
   - GC time: allocation churn during startup
Trace each suspicious counter to source code with one diagnostic load. These flags perturb
   timing, so use them only to attribute:
   - --log-ic: which source positions miss, and their map states (polymorphic, megamorphic)
   - --log-maps: which objects create most transitions; check whether a shared root shape
     (plain {}) fills up or hot objects go to dictionary mode
   - --log-function-events: which functions compile lazily, how big they are, when they first run
   The sampler books lazy-compile time on the calling frame, so a wrapper that looks hot may
   just be compiling its callee.
Root-cause in the engine source. When a counter or log entry is surprising, read the V8,
   Blink or Chromium code that produces it, at the matching version. Don't rely on blog posts
   or memory, because engine behaviour changes between versions. Typical questions:
   - What does this RCS counter or log event actually measure, and what triggers it?
   - Which limit or heuristic is being hit (transition caps, dictionary-mode thresholds,
     literal boilerplate limits, lazy-compile and PIFE heuristics, stack capture depth)?
   - Why does an IC go megamorphic, or a protector get invalidated, here?
   - Which DevTools protocol call or automation default changes engine behaviour?
   Where you find a root cause, give a cogent step-by-step explanation with references to the
   relevant Chromium, V8 or Blink source code, if that structure suits the finding. Go from the
   page's code, through each engine step, to the measured cost. At each step cite the file,
   function and constant (for example src/objects/transitions.h, kMaxNumberOfTransitions) so the
   engineer can follow and check it. Where it's cheap, confirm the reading with a small repro in
   Node or d8 at the same V8 version.
Count suspicious runtime events by instrumenting the bundle if needed: module factories,
   require calls, getter calls on re-exports, defineProperty calls, late prototype writes.
   Large counts relative to the size of the app usually mean bundler or transpiler glue, which
   can be fixed in the build config instead of app code.
Before suggesting a build change, check what the bundler emitted (module wrappers, export
   getters, polyfills pulled in by usage detection, functions the minifier inlined into giants).
For each candidate, estimate its upper bound cheaply: patch the bundle or switch the pattern
   off, then rerun the RCS load and see how far the counter and its time drop.
Report a short ranked list. For each item: the counter and its value, the source pattern behind
it (file or code shape), the root-cause explanation from step 3 with its source references, the
fix, the estimated saving in ms with how you estimated it, and how confident you are. Mark
anything seen in only one load, and any engine explanation not checked against the source, as
unconfirmed.
