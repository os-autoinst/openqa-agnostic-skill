---
name: openqa-agnostic
description: >
  Convert an existing openQA test module into a standalone
  "openqa-agnostic" test that also runs outside openQA, or propose a
  prioritized list of which modules are worth converting. Works for any
  test domain (security, console, etc.). Use when asked to "make this
  test agnostic", "port this to openqa-agnostic", "convert this test to
  run standalone", "what tests can we convert", or "list openqa-agnostic
  candidates".
---

# openQA -> openqa-agnostic conversion

Turn an openQA test module into a portable test artifact (bash/python/go/
java) that runs both inside openQA (thin `.pm` wrapper) and standalone on
any SUT with the runtime installed. Reference conversions exist across
multiple domains:

**Security** (7 conversions): `testPolkit`, `testPostQuantumCrypto`,
`testApacheSSLPQC`, `goPostQuantum`, `java_hashing`, `yama`,
`testOqsProvider` -- under `data/security/openqa_agnostic/<lang>/<name>/`
and `tests/security/oqa_agnostic/<name>.pm`.

**Console** (1 conversion): `testLibsoup` -- under
`data/console/openqa_agnostic/python/testLibsoup/` and
`tests/console/oqa_agnostic/libsoup.pm`.

Read the matching files before writing anything new; they are the ground
truth, this document is a map of them.

## 0. Two invocation modes

- **No specific module named** ("what can we convert?", "list candidates",
  or the skill invoked with no target) → run **Phase A: Discovery** below,
  present the prioritized list, and stop  -- wait for the user to pick one
  before converting anything.
- **A specific module named** ("convert tests/security/x/y.pm") → skip
  straight to **Phase B: Conversion**, starting with the perimeter check.

Always start a fresh session's first real request with Discovery unless the
user already named a target module  -- it prevents wasted effort on a module
that turns out to be disqualified.

## 1. Perimeter (shared by both phases)

The agnostic pattern now works across domains. The runner
(`lib/agnosticTestRunner.pm`) accepts a `domain` arg that controls
which `data/` subtree test files live in. Currently supported domains:

- **security**: `tests/security/`, `tests/fips/`,
  `data/security/openqa_agnostic/`, `tests/security/oqa_agnostic/`
- **console**: `tests/console/`, `data/console/openqa_agnostic/`,
  `tests/console/oqa_agnostic/`

For Discovery, scope the search to the relevant domain's test paths.
For Conversion, confirm the target module lives in a supported domain.

## 2. Phase A  -- Discovery: enumerate and rank candidates

Goal: turn 200+ test modules into a short, prioritized "convert this next"
list. This requires actually reading each candidate and reasoning about
what it tests  -- **not** a keyword/regex scoring pass. An earlier version of
this skill classified modules by counting calls to a fixed set of function
names (`mutex_lock`, `barrier_wait`, ...). That missed `mutex_wait`  -- a real
function used by `tests/security/cc/ipsec/ipsec_client.pm` and
`tests/security/mariadb/mariadb_ssl_client.pm` to block on a second SUT  --
so both were mis-classified as safe standalone candidates when they are
genuinely multi-machine. A fixed keyword list will always miss the next
variant; understanding what a module does does not have that failure mode.

1. **Enumerate.** List every `.pm` under the target domain's test paths,
   then drop anything that already uses the agnostic runner (don't rely
   on the `oqa_agnostic/` directory name alone, grep the `use` line so
   the check stays correct as the library grows):

   ```bash
   # Security domain:
   find tests/security tests/fips -name '*.pm' | \
     xargs grep -L 'use.*agnosticTestRunner' > /tmp/candidates.txt
   # Console domain:
   find tests/console -name '*.pm' | \
     xargs grep -L 'use.*agnosticTestRunner' > /tmp/candidates.txt
   ```

2. **Use cheap signals only to order the reading queue, never as the
   verdict.** A quick grep pass for suspicious tokens (any `mutex_`/
   `barrier_`/`parallel_` call, `PARALLEL_WITH`, paired IP vars like
   `SERVER_IP`/`CLIENT_IP`, `assert_screen`/`check_screen`/
   `wait_screen_change`/`wait_still_screen`, an `installbasetest` base
   class, `*_setup`/`*_prepare`/`*_env`/`*_baseline` naming, a lone
   `milestone => 1`) is fine for deciding which files are likely to need
   the harder judgment calls and which are likely quick "yes, portable"
   reads  -- but a hit or a miss on any of these is a hint, not an answer.
   The `mutex_wait` miss above is the concrete reason why.

3. **Read every candidate** (its full body  -- plus one hop into any
   `lib/security/*.pm` or other shared helper it calls, and the matching
   `schedule/**/*.yaml` entry if machine coordination is suspected) and
   answer these questions from what's actually there:

   - **Multi-machine?** Does it block on state produced by a different SUT
      -- any wait/sync call on a mutex or barrier under any name, a
     `PARALLEL_WITH` job-group setting, or two distinct machine-role IP
     vars? If yes, don't disqualify yet  -- go to the next question.
   - **Collapsible multi-endpoint?** If it's a client/server pair, is what's
     being verified protocol/config correctness (→ **collapsible**: both
     roles can run from the single agnostic script on one host, isolated
     with a container or network namespace if they need distinct
     addresses  -- precedent: `testApacheSSLPQC` and `goPostQuantum`, which
     already spin up server *and* client from the same script on
     localhost), or does the test depend on two genuinely separate
     hosts/kernels/NICs (real routing, firewall traversal, a physical
     network path → **not collapsible**, stays disqualified as
     multi-machine)? This is a judgment call about what property the test
     is actually verifying  -- make it explicitly, don't default either way.
   - **GUI/needle-dependent?** Are `assert_screen`/`check_screen`/etc. calls
     incidental (e.g. only to reach a login prompt, trivially replaced by
     `select_serial_terminal`) or load-bearing (verifying a GUI dialog, an
     X11 app, a visual state that has no serial-console equivalent)?
   - **Bootstrap/propaedeutic?** Does the module itself end with real
     assertions of an outcome, or does it only install/configure/reboot and
     leave the checking to a sibling module in the same scenario? (An
     `installbasetest` base class or `*_setup`/`*_prepare`/`*_env`/
     `*_baseline` name is a strong hint this is the case  -- confirm by
     reading, don't stop at the name.)
   - **Delegated logic?** Does the real work live in a `lib/security/*.pm`
     (or other shared lib) helper the module calls? If so, read that helper
     too  -- a wrapper's own body can look trivially portable while the
     helper it calls is needle-heavy or multi-machine.
   - **Reboot-path needle dependency?** Does the module call
     `power_action('reboot', ...)` + `$self->wait_boot(...)`, or
     transactional's `process_reboot`? If so, don't stop at the module's own
     body  -- `wait_boot` (`lib/opensusebasetest.pm`), `power_action`
     (`lib/power_action_utils.pm`), and `process_reboot`
     (`lib/transactional.pm`) are themselves built on `assert_screen`/
     `check_screen` against GRUB-menu, login-prompt, and `tianocore-mainmenu`
     needles on any non-serial-console backend. A module with zero needles
     of its own can still be needle-dependent through this one call. This is
     the single most common blind spot found so far  -- a first classification
     pass checked modules' own source for needles and missed it in 10 of 16
     modules later confirmed to reboot mid-test.

4. **Classify** from that reading, one of:
   - **Disqualified  -- multi-machine.** Genuinely needs two coordinating
     SUTs and isn't collapsible.
   - **Excluded  -- bootstrap/propaedeutic.** Sets up state for a sibling
     module; nothing to run standalone.
   - **Not convertible  -- GUI/needle-core.** Core assertions depend on
     screen matching with no serial-console equivalent  -- including a
     reboot-path needle dependency (previous bullet) where the module's
     *methodology* itself requires reboot-and-recheck cycles. Note this
     specific case is salvageable with design work (replace needle-based
     `wait_boot` with a portable boot-detection loop  -- poll for SSH/serial
     reachability instead of matching a login-prompt needle)  -- if you judge
     that's straightforward for a given module, say so in the rationale
     instead of a flat disqualification, the same way `collapsible_multi_endpoint`
     is called out rather than silently merged into "Candidate."
   - **Collapsible multi-endpoint.** Client/server today, but convertible
     with same-host/container/netns isolation  -- call this out as its own
     tier, don't silently fold it into plain "Candidate": it needs upfront
     design work (which side runs first, how they find each other on
     localhost) that a pure single-role module doesn't.
   - **Candidate.** Fully expressible through `assert_script_run`/
     `script_run`/`script_output`/`validate_script_output` on one host, no
     needles, nothing delegated to a non-portable helper.

   Every verdict carries a one-sentence rationale that cites what was
   actually read (a specific call, a schedule entry, an assertion)  -- a
   bucket assignment with no rationale isn't a finished classification.

5. **Self-verify every verdict before it's final  -- not just the positive
   ones.** One more pass, done by you, serially, not a second agent.
   Re-read the same module specifically trying to REFUTE your own verdict,
   whichever bucket it landed in: re-check the reboot-path needle question
   above, grep `schedule/**/*.yaml` for the module's short name for a
   `PARALLEL_WITH` you might have missed, and re-open any delegated helper
   with fresh eyes. A first version of this step only re-checked
   `candidate`/`collapsible_multi_endpoint` verdicts (163 of 243) and caught
   16 wrong ones (~10%)  -- mostly the reboot/`wait_boot` blind spot above.
   But a later audit of the *un*-verified negative buckets
   (disqualified-mm/excluded-bootstrap/not-convertible-gui) found a real
   borderline case there too: a module classified `excluded_bootstrap`
   whose body ends in a genuine multi-field `validate_script_output` check
   (not just "did my config write succeed")  -- arguably a candidate, or at
   least not the clear-cut "asserts nothing" the summary line claimed.
   Only checking the positive bucket structurally can't catch a module that
   was wrongly excluded  -- nobody re-examines a verdict that already reads
   "nothing to see here." Self-verify applies to every verdict, negative
   included.
   - Specifically for `excluded_bootstrap`: a nonzero `assertion_count`
     does not automatically mean it's not bootstrap (a setup module can
     legitimately call `assert_script_run` many times just to confirm its
     own config-writes succeeded)  -- but before excluding, check whether any
     of those assertions verify a *security-relevant outcome* rather than
     "did the command exit 0" (e.g. a detailed `sestatus`/config-dump regex
     confirming the target state was actually reached, not just written).
     If so, don't flatten that nuance away in the one-line summary you
     present later  -- say explicitly "has real assertions, excluded because
     X" rather than "asserts nothing."

6. **Rank the Candidate (and Collapsible) tiers** before presenting them:
   - Deprioritize tiny fragments (rule of thumb: well under ~40 lines and
     one or two assertions) that read as one step of a larger multi-module
     scenario rather than a complete check  -- list separately as "scenario
     fragments," don't drop them silently.
   - Prioritize modules with real assertion density (several
     `assert_script_run`/`validate_script_output` calls) and a
     self-contained scope  -- read like `tests/security/usbguard/usbguard.pm`
     (the calibration example: 144 lines, zero needles, zero machine
     coordination, pure assert/validate_script_output).
   - Cheaper conversions first within a tier, so the proposal front-loads
     quick wins.

7. **Present the result** as a short report, not a data dump:
   - One line per module in the top-ranked tier: path, a short rationale
     grounded in what was read, and rough size.
   - A separate short list for "Collapsible multi-endpoint" candidates,
     since they need a design note (how the two roles get isolated) before
     anyone starts scaffolding.
   - A rollup count for every other bucket (disqualified-mm,
     bootstrap-excluded, not-convertible-needle, scenario-fragments)  -- don't
     enumerate all of them, just say how many and offer to list a bucket on
     request.
   - **Scale note  -- serial only, no parallel agents, no Workflow tool.**
     This phase means opening on the order of 200 files, each with its own
     self-verify pass (§2 step 5)  -- that's real, slow work, by design. Do
     not fan this out with the Workflow tool or parallel subagents, even if
     the user doesn't object: a full sweep costs on the order of 400+ agent
     calls and tens of millions of tokens, which is not an acceptable
     default cost for a discovery pass. Process candidates **one at a time,
     in a loop, in a single thread of execution**:
     - Read the file (+ any delegated helper, + schedule entry if machine
       coordination is suspected), form a verdict with rationale, self-verify
       it (§2 step 5) if it's a `candidate`/`collapsible`, then immediately
       append one line (path, verdict, one-line rationale) to a running
       scratch file on disk  -- don't hold the full source of prior files in
       context, only the accumulated verdict lines.
     - This means the conversation's context fills with verdicts, not raw
       source, so a 200+ file sweep stays tractable even as earlier turns
       get compressed away  -- the scratch file on disk is the durable record,
       not your own memory of having read file #12.
     - For a narrow ask ("what's convertible under
       `tests/security/selinux/`?") this is fast (a couple dozen files).
       For a full-repo sweep, say up front that it will take a while and
       proceed  -- don't ask for permission to go slow, only flag it if the
       user seems to expect an instant answer.
   - An explicit caveat: even a fully-reasoned "Candidate" can still turn
     out unsuitable once conversion starts (coupling to a sibling module
     that only becomes obvious mid-port, a helper three hops deep that
     turns out needle-heavy)  -- always re-confirm in Phase B step 1 before
     scaffolding anything, even for top-ranked candidates.

## 3. Phase B  -- Conversion

### 3.1 Disqualify before doing any other work

Apply the same reasoning as Phase A §2–3 to the named target: read it fully
(and its called helpers / matching schedule entry), and check for
multi-machine coordination, GUI/needle dependence, and whether a
client/server shape is collapsible to one host. **Multi-machine tests that
aren't collapsible cannot be made agnostic**  -- stop immediately and explain
why. Also disqualify modules whose core assertions depend on needle/screen
matching, VNC/GUI interaction, or physical hardware presence (e.g. TPM
button press)  -- there's no way to express those outside openQA's video
backend. A good candidate does everything through
`select_serial_terminal`/`select_console` plus `assert_script_run` /
`script_run` / `script_output` / `validate_script_output`  -- no needles, no
unavoidable second SUT.

### 3.2 Architecture (what exists, where it goes)

- **`lib/agnosticTestRunner.pm`** -- generic OO helper:
  `new({language, name, domain})->setup->run_test->parse_results->cleanup`.
  - `language` must be exactly `go`, `python`, or `java` -- the constructor
    `die`s otherwise. Never invent another value.
  - `domain` controls which `data/` subtree test files live in. Defaults
    to `'security'`. Use `'console'` for console tests, etc.
  - `name` must equal the directory name under
    `data/<domain>/openqa_agnostic/<lang>/<name>/` exactly -- it drives
    both `data_url_path` and `test_dir` (default `~/<name>`).
  - `result_format`/`result_file` are derived automatically from
    `language` (TAP for java, XUnit otherwise) -- don't override them.
  - `helper_path` defaults to `openqa_agnostic/lib/helper.sh` (shared
    across all domains). Override only if you have a domain-specific
    helper.
  - `skip_phub` (default 0) -- set to 1 to skip PackageHub registration
    on SLE < 16 when it's not needed.
  - `setup()` zypper-installs the language runtime, downloads test files
    from `data/<domain>/openqa_agnostic/<lang>/<name>/`, and downloads
    `helper.sh` from the shared location. It discovers which files to
    fetch by running `./runtest -f` on the SUT.
  - `run_test()` runs `./runtest`, then moves `results.xml`/`results.tap`
    to `result_file`, then resets the terminal (a broken-echo workaround).
  - `cleanup()` does `cd ~ && rm -rf $test_dir`. The `cd ~` is required
    because `run_test()` leaves the shell cwd inside `test_dir` -- without
    it, subsequent commands fail with "getcwd: cannot access parent
    directories" when the runner is called in a loop (e.g.
    `fips_java_crypto.pm` runs two tests sequentially).
  - `cleanup()` does **not** restore application/system state -- any state
    teardown (restore a hostname, delete a test user, remove a temp polkit
    rule) must happen inside the test itself (trap/finally/pytest fixture
    teardown).

  **Backward compatibility**: `lib/security/agnosticTestRunner.pm` still
  exists as a thin wrapper that delegates to the generic runner with
  `domain => 'security'`. Existing code using `security::agnosticTestRunner`
  continues to work unchanged.

- **`data/<domain>/openqa_agnostic/<lang>/<TestName>/`** -- the portable
  artifact: language source file(s) + a `runtest` script + configuration file(s), if any.
  Any configuration file(s) needed by the test must not be inline in the test
  code, prefer separate configuration file(s). They must reside in
  `data/<domain>/openqa_agnostic/<lang>/<TestName>/`.
  Examples: `data/security/openqa_agnostic/python/testOqsProvider/oqs-openssl.cnf` or
  `data/security/openqa_agnostic/python/testApacheSSLPQC/pqc-ssl.conf`
  Canonical templates: `testPolkit/runtest` (go), `yama/runtest` (python, with the
  `set -e`-relaxation trick), and `testLibsoup/runtest` (python, wrapping
  an external test runner). Every `runtest` has this shape:

  ```bash
  #!/bin/bash
  # METADATA_START
  # ---
  # test: <short name>
  # desc: <one line>
  # steps:
  #   - ...
  # author: <email>
  # maintainer: <name and email -- see note below>
  # expected: <what "pass" looks like>
  # platform: <e.g. SLE16, Tumbleweed>
  # tags: <domain> <keywords> <poo#/bsc# ref if any>
  # ---
  # METADATA_END

  set -e
  source ../lib/helper.sh
  TEST_FILES=(<every file besides runtest that setup() must download>)
  handle_args "$0" "$@"

  # actual invocation, e.g.:
  pytest -v <file>.py --junitxml=results.xml       # python, -v for verbose output
  gotestsum --format=standard-verbose --junitfile results.xml   # go, after `go clean -testcache`
  javac X.java && java X > results.tap          # java, hand-rolled TAP (no JUnit runner on SUT)
  ```

  `handle_args` (from `lib/helper.sh`) implements `-h/--help`,
  `-m/--metadata` (dumps the METADATA block), and `-f/--files` (prints
  `TEST_FILES` space-separated)  -- `agnosticTestRunner::setup()` calls `-f`
  to know what to fetch, so `TEST_FILES` must be complete and accurate.

  For pytest-based tests, a non-zero pytest exit code must still leave
  `results.xml` on disk and exit 0 from `runtest`  -- the failure surfaces
  later via the uploaded XML, not via the script's own exit code. Wrap the
  pytest call: `set +e; pytest ...; rc=$?; set -e; [ -f results.xml ] &&
  exit 0; exit $rc` (see `yama/runtest`).

  For pytest-based tests, the resulting python code MUST BE compatible with Python 3.6,
  so do not use Python 3.7+ only function invocations and features. This
  matters because SLE 15 ships Python 3.6 and SLE 12 ships Python 3.4.
  f-strings and `subprocess.run` both require Python 3.6+; using them is
  fine and idiomatic, but they will cause a collection-time `SyntaxError`
  crash on SLE 12, producing a hard module failure instead of a skip.

  **Platform version guard -- required when the test's schedule reaches
  SLE 12 or any other platform with Python < 3.6.** Check the
  `conditional_schedule` sections of every YAML that includes the new
  module: if any branch reaches a product with Python < 3.6 (SLE 12-SP5,
  SLE 12-SP3), add a version guard at the top of `runtest` (after
  `ensure_root`) that skips gracefully rather than crashing at collection:

  ```bash
  # Skip on Python < 3.6 (SLE 12 ships 3.4; f-strings and subprocess.run
  # require 3.6+). Emit a JUnit skip result so the module reports clean.
  py_ver=$(python3 -c "import sys; print('%d%02d' % sys.version_info[:2])" \
           2>/dev/null || echo "000")
  if [ "$py_ver" -lt 306 ]; then
      echo "SKIP: Python 3.6+ required (found $(python3 --version 2>&1))"
      cat > results.xml <<EOF
  <?xml version="1.0" encoding="UTF-8"?>
  <testsuite name="<TestName>" tests="1" skipped="1" failures="0" errors="0">
    <testcase name="<test_name>" classname="<TestName>">
      <skipped message="Python 3.6+ required for f-strings and subprocess.run"/>
    </testcase>
  </testsuite>
  EOF
      exit 0
  fi
  ```

  If the module is explicitly scoped to SLE 15+ or newer in its `platform:`
  metadata and the schedule only reaches those versions, the guard is not
  needed -- but verify the schedule before omitting it. The concrete failure
  mode (a collection-time `SyntaxError` on Python 3.4 crashing the entire
  `pytest` run rather than skipping) was first observed in `testSudo` on
  SLE 12-SP5 s390x via `mau-extratests2.yaml`, where `sudo_agnostic` sits
  in the unguarded `schedule:` section that runs on all versions including
  12-SP5. See os-autoinst-distri-opensuse PR #26742 for the fix.

  DO NOT install packages under test in `runtest`

  List contents of test execution directory to stdout using
  ```
  echo "--- Test directory: $PWD ---"
  ls -l
  echo "----------------------------"
  ```

  Use bash command `install` to install the configuration file(s) and cat their
  contents to the stdout as in `data/security/openqa_agnostic/python/testOqsProvider/runtest`

- **`data/openqa_agnostic/lib/helper.sh`** -- shared across all domains.
  Provides `handle_args`, `ensure_root`, `ensure_command_available`.
  Source it, never duplicate its logic into a new `runtest`. There is
  one copy, not one per domain.

- **`tests/<domain>/oqa_agnostic/<name>.pm`** -- thin openQA wrapper:

  ```perl
  use Mojo::Base 'opensusebasetest';
  use testapi;
  use serial_terminal 'select_serial_terminal';
  use package_utils 'install_package';
  use agnosticTestRunner;

  sub run {
      select_serial_terminal;
      # Package installation belongs here, not in runtest.
      # Use install_package with trup_continue for immutable system support.
      install_package('foo-tests gnome-desktop-testing', trup_continue => 1);
      my $test = agnosticTestRunner->new({
              language => 'python',
              name => '<TestName>',
              domain => '<domain>',
          }
      );
      $test->setup()->run_test()->parse_results()->cleanup();
  }
  1;
  ```

  Use `install_package` with `trup_continue => 1` instead of
  `zypper_call` so the wrapper works on both regular and
  transactional/immutable systems.

  One codebase exception, `go_post_quantum.pm`, bypasses the runner
  entirely for a simple stdout-substring check with no XUnit/TAP result --
  that's a legacy shortcut, not the pattern to copy for new conversions.

  Do not use
  ```
  sub test_flags {
     return {always_rollback => 1};
  }
  ```
  if the test is to be used / scheduled on architectures not supporting rollback
  such as s390x. Ask user about targeted architectures.

  Record package(s) under test version(s) using `record_info`, like
  ```
    my $<package>_version = script_output(q(rpm -q --queryformat '%{VERSION}' <package>));
    record_info('<package>', "version $<package>_version");
  ```

- **Scheduling**: the module is referenced by its openQA path
  (`<domain>/oqa_agnostic/<name>`) from a `schedule/<domain>/*.yaml`
  file (e.g. `schedule/security/yama.yaml`,
  `schedule/console/libsoup_installed_tests.yaml`).

### 3.3 Conversion workflow

1. Read the source `.pm` fully and classify every statement:
   - **Portable**  -- `assert_script_run`, `script_run`, `script_output`,
     `validate_script_output`  -- map 1:1 onto shell/python/go: run a
     command, check exit code, regex-match output.
   - **openQA-only, keep in the wrapper**  -- `select_console`/
     `select_serial_terminal`, `zypper_call`/`install_package`,
     `power_action`+`wait_boot`, `record_info`/`record_soft_failure`,
     `is_sle`/`is_tumbleweed` version gating. If trivial (e.g. "is this
     binary installed"), it may instead become a portable runtime
     check/skip inside the agnostic test  -- judgment call, prefer keeping
     version/environment gating in the `.pm` since it's already openQA-only
     context.
   - **Collapsible multi-endpoint** (if flagged in Discovery or found here):
     design how both roles run from the one agnostic script on a single
     host  -- e.g. spawn the server as a background process/goroutine and
     have the client connect over `localhost`, or use a network namespace /
     container per role if they need genuinely distinct addresses. Document
     the chosen approach in the `runtest` METADATA `desc`/`steps`.
2. Pick the language: **`agnosticTestRunner` only accepts `go`/`python`/
   `java`  -- there is no standalone "bash" option**, since the runner needs
   one of these to install a runtime and produce XUnit/TAP results; a
   `runtest` script is always bash, but the test *body* it invokes must be
   one of the three. Default to python+pytest for plain CLI-driven checks
   (`subprocess.run` + `assert`/regex  -- least boilerplate, no compile step,
   see `data/security/openqa_agnostic/aa_disable/test_aa_disable.py` for a
   minimal single-assertion example); go for
   native crypto/TLS/networking (reuse `testPolkit/utils.go`'s
   `RunCommandTimeout` instead of reinventing subprocess+timeout handling);
   java only for JVM/JCA-specific testing.
3. Scaffold `data/<domain>/openqa_agnostic/<lang>/<TestName>/`: source
   file(s) + `runtest` per the template in §3.2, complete METADATA block.
   Ask the user for `author` if not inferable from `git config user.email`
   or recent commits -- don't guess a name. For `maintainer`, ask the
   developer: use the team/squad name (e.g. `QE Security <none@suse.de>`)
   only if the squad has agreed to own the test. Otherwise use the
   individual developer's name and email. The same applies to the
   `# Maintainer:` comment in the `.pm` wrapper.
4. Write `tests/<domain>/oqa_agnostic/<name>.pm` per the wrapper template
   in §3.2, porting the openQA-only pre-steps found in step 1 verbatim
   where possible. Use `install_package` with `trup_continue => 1` for
   package installation. Preserve the original `test_flags` unless there's
   a clear reason to change it -- flag it if unsure.
5. Wire scheduling: add/update the `<domain>/oqa_agnostic/<name>` entry in
   the appropriate `schedule/<domain>/*.yaml` file.
6. Flag the old `.pm` module (and any now-orphaned schedule entry) for
   removal. **Ask for explicit confirmation before deleting**  -- other YAML
   files or job groups may still reference it; grep for the module's short
   name across `schedule/` first.
7. Validate what's checkable without a full SUT: `./runtest -h`,
   `./runtest -m`, `./runtest -f` should all work standalone since they
   only depend on `helper.sh`, not the target system. State explicitly that
   the test body itself still needs a real SUT run to confirm it passes  --
   never claim a ported test passes without having run it.

## 4. Hard constraints

- `language` is only `go|python|java` in `agnosticTestRunner->new()`.
- `domain` must match a supported domain (`security`, `console`, etc.).
- `name` must match the `data/<domain>/openqa_agnostic/<lang>/<name>`
  directory exactly.
- Don't override `result_format`/`result_file` -- they're derived from
  `language`.
- `helper.sh` lives at `data/openqa_agnostic/lib/helper.sh` (one copy,
  shared across all domains). It is sourced as `../lib/helper.sh` on the
  SUT because `setup()` downloads it to a sibling `lib/` dir one level
  above `test_dir`. Don't change that layout, don't duplicate helper.sh
  per domain.
- Package installation belongs in the `.pm` wrapper, not in `runtest`.
  Use `install_package` with `trup_continue => 1` for immutable system
  support.
- Never classify a module as convertible (or not) from a keyword/regex
  scan alone -- read it (§2-3.1) before rendering a verdict.
- Never use the Workflow tool or parallel subagents for Discovery --
  process candidates serially, one at a time (§2 step 7 scale note).
  A full-repo sweep is slow by design; that's the accepted tradeoff
  for bounded cost.
