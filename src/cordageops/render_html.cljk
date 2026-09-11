(ns cordageops.render-html
  "Build-time HTML renderer for `docs/samples/operator-console.html`.

  Closes flagship checklist item 2 for this repo: it previously shipped
  no demo page and no generator at all. This namespace drives the REAL
  actor stack -- `cordageops.operation/build` (the langgraph-clj
  StateGraph) -> `cordageops.advisor` -> `cordageops.governor` ->
  `cordageops.phase` -> `cordageops.registry` -> `cordageops.store` --
  and renders the resulting SSoT + append-only audit ledger.

  EVERY id, number, disposition and hold reason on the generated page is
  actual output of that run. Nothing is hand-typed: the batch/equipment
  rows are `cordageops.store`'s own records after the scenario mutated
  them, the draft numbers (`MNT-000000`, `SHP-000000`) are
  `cordageops.registry`'s own sequence output, and every ledger row is a
  fact the `:commit` or `:hold` node actually appended. The ONE
  hand-written table is the action gate, which documents this actor's
  fixed op contract (README `Ops`, `cordageops.governor`/
  `cordageops.phase`) rather than any runtime measurement -- it is
  labelled as such on the page.

  Subject ids come from `cordageops.store/sample-data!` and were
  cross-checked against it before use (`batch-001`..`batch-003`,
  `equip-001`, `equip-002`); the maintenance/safety/shipment subjects
  (`mnt-*`, `concern-1`, `ship-*`) are ids this scenario itself creates,
  exactly as `cordageops.sim` does.

  DETERMINISTIC: no timestamps, no randomness, no unordered-map
  iteration -- two consecutive runs are byte-identical (verify by
  diffing the output of two `clojure -M:dev:render-html` invocations).

  Usage: `clojure -M:dev:render-html [out-file]`
  (default `docs/samples/operator-console.html`)."
  (:require [jp-go-dds.skin]
            [kotoba.lang.text :as str]
            [cordageops.store :as store]
            [cordageops.operation :as op]
            [langgraph.graph :as g]))

(def ^:private coordinator
  {:actor-id "coord-1" :actor-role :plant-coordinator :phase 3})

(defn- exec! [actor tid request]
  (g/run* actor {:request request :context coordinator} {:thread-id tid}))

(defn- approve! [actor tid]
  (g/run* actor {:approval {:status :approved :by "coord-1"}}
          {:thread-id tid :resume? true}))

(defn run-demo!
  "Seeds a fresh `cordageops.store/mem-store` with
  `cordageops.store/sample-data!`, builds the REAL
  `cordageops.operation` actor over it, and drives one scenario that
  reaches every disposition this actor can produce.

  ONE FULL CLEAN LIFECYCLE (batch-001 / equip-001):
    1. `:log-production-batch` batch-001 with a clean patch -- the only
       op in phase 3's `:auto` set, so it auto-commits governor-clean
       with no human in the loop.
    2. `:schedule-maintenance` mnt-1 against equip-001 (verified AND
       registered twisting machine) -- `cordageops.phase` never puts
       `:schedule-maintenance` in any phase's `:auto` set, so a clean
       verdict still escalates; a human plant supervisor approves and it
       commits draft MNT-000000.
    3. `:flag-safety-concern` concern-1 -- always `:stake
       :coordination/safety-concern`, so the governor escalates
       regardless of confidence; approved and committed to the
       safety-concern log.
    4. `:coordinate-shipment` ship-1 against batch-001 for 5000 m
       (within its own recorded 20000 m production length) -- escalates,
       approved, commits draft SHP-000000 and pushes batch-001's own
       cumulative shipped length 5000 -> 10000 m.

  HARD GOVERNOR HOLDS (each one is rejected at `:decide` and routed
  straight to `:hold` -- NO human is ever asked, no phase and no
  approver can override):
    5. `:line-operate-blocked`      -- mnt-3 asks to DIRECTLY OPERATE
                                       equip-001 (`:direct-operate?
                                       true`) instead of drafting a
                                       maintenance window. This actor's
                                       permanent scope boundary.
    6. `:equipment-not-verified`    -- mnt-2 targets equip-002, an
                                       UNVERIFIED/unregistered braiding
                                       machine.
    7. `:batch-not-verified`        -- ship-2 targets batch-003, an
                                       UNVERIFIED/unregistered batch.
    8. `:shipment-volume-exceeded`  -- ship-3 asks for 1000 m off
                                       batch-002, whose own record
                                       already shows 5700 of 6000 m
                                       shipped; the governor recomputes
                                       the headroom itself.
    9. `:already-scheduled`         -- mnt-1 again, after step 2 already
                                       committed it.
   10. `:invalid-breaking-strength` -- a batch-002 patch declaring a
                                       999999 kN breaking-strength-test
                                       reading.
   11. `:invalid-grade`             -- a batch-003 patch declaring a
                                       fabricated `:premium-plus-select`
                                       quality grade.
   12. `:not-propose-effect`        -- a mis-wired caller whose own
                                       request `:effect` is
                                       `:direct-write`, not `:propose`.

  Returns the store. Every field `render` reads below is real
  governor/store output."
  []
  (let [db (-> (store/mem-store) (store/sample-data!))
        actor (op/build db)]

    ;; --- clean lifecycle -------------------------------------------------
    (exec! actor "t1-log"
           {:op :log-production-batch :effect :propose :subject "batch-001"
            :patch {:quality-grade :grade-a :last-assessed "2026-07-14"}})

    (exec! actor "t1-maint"
           {:op :schedule-maintenance :effect :propose :subject "mnt-1"
            :value {:equipment-id "equip-001" :maintenance-type :spindle-inspection
                    :scheduled-date "2026-08-01" :direct-operate? false}})
    (approve! actor "t1-maint")

    (exec! actor "t1-safety"
           {:op :flag-safety-concern :effect :propose :subject "concern-1"
            :value {:equipment-id "equip-001" :concern-type :equipment-safety
                    :severity :moderate
                    :description "撚糸機のガード固定に緩みを確認"}})
    (approve! actor "t1-safety")

    (exec! actor "t1-ship"
           {:op :coordinate-shipment :effect :propose :subject "ship-1"
            :value {:batch-id "batch-001" :length-meters 5000.0
                    :destination "buyer-warehouse-north"}})
    (approve! actor "t1-ship")

    ;; --- HARD holds ------------------------------------------------------
    (exec! actor "t-direct-operate"
           {:op :schedule-maintenance :effect :propose :subject "mnt-3"
            :value {:equipment-id "equip-001" :maintenance-type :emergency-run
                    :scheduled-date "2026-09-01" :direct-operate? true}})

    (exec! actor "t-unverified-equip"
           {:op :schedule-maintenance :effect :propose :subject "mnt-2"
            :value {:equipment-id "equip-002" :maintenance-type :tension-calibration
                    :scheduled-date "2026-08-01" :direct-operate? false}})

    (exec! actor "t-unverified-batch"
           {:op :coordinate-shipment :effect :propose :subject "ship-2"
            :value {:batch-id "batch-003" :length-meters 1000.0
                    :destination "buyer-warehouse-south"}})

    (exec! actor "t-over-volume"
           {:op :coordinate-shipment :effect :propose :subject "ship-3"
            :value {:batch-id "batch-002" :length-meters 1000.0
                    :destination "buyer-warehouse-east"}})

    (exec! actor "t-double-schedule"
           {:op :schedule-maintenance :effect :propose :subject "mnt-1"
            :value {:equipment-id "equip-001" :maintenance-type :spindle-inspection
                    :scheduled-date "2026-08-01" :direct-operate? false}})

    (exec! actor "t-bad-strength"
           {:op :log-production-batch :effect :propose :subject "batch-002"
            :patch {:breaking-strength-kn 999999.0}})

    (exec! actor "t-bad-grade"
           {:op :log-production-batch :effect :propose :subject "batch-003"
            :patch {:quality-grade :premium-plus-select}})

    (exec! actor "t-miswired-caller"
           {:op :log-production-batch :effect :direct-write :subject "batch-001"
            :patch {:quality-grade :grade-a}})

    db))

;; ----------------------------- rendering -----------------------------

(defn- esc [v]
  (-> (str v)
      (str/replace "&" "&amp;")
      (str/replace "<" "&lt;")
      (str/replace ">" "&gt;")
      (str/replace "\"" "&quot;")))

(defn- kw-name
  "`name` for keywords/strings, safe on nil and on namespaced keywords
  (keeps the namespace so `:safety-concern/flag` stays readable)."
  [v]
  (cond
    (nil? v) ""
    (keyword? v) (subs (str v) 1)
    :else (str v)))

(defn- num->s
  "Deterministic number rendering: whole doubles lose the `.0` tail, so
  the page reads `20000` rather than `20000.0`. No locale, no rounding
  beyond that."
  [v]
  (cond
    (nil? v) "—"
    (and (number? v) (== (double v) (Math/rint (double v)))) (str (long v))
    :else (str v)))

(defn- flag-cell [ok? ok-label bad-label]
  (if ok?
    (str "<span class=\"ok\">" ok-label "</span>")
    (str "<span class=\"critical\">" bad-label "</span>")))

(defn- status-cell
  "Roll-up of every append-only ledger fact whose `:subject` is
  `subject` -- how many the `:commit` node wrote, how many the `:hold`
  node wrote, and which governor rules did the holding. Derived
  entirely from the log; nothing is assumed about ops that produced no
  fact."
  [ledger subject]
  (let [fs (filter #(= (:subject %) subject) ledger)
        committed (count (filter #(= :committed (:t %)) fs))
        held (filter #(= :governor-hold (:t %)) fs)
        rules (distinct (mapcat #(map :rule (:violations %)) held))]
    (if (empty? fs)
      "<span class=\"muted\">no ledger activity</span>"
      (str/join " &middot; "
                (cond-> []
                  (pos? committed)
                  (conj (str "<span class=\"ok\">" committed " committed</span>"))

                  (seq held)
                  (conj (str "<span class=\"critical\">" (count held) " HARD hold"
                             (when (> (count held) 1) "s") " &middot; "
                             (esc (str/join ", " (map kw-name rules)))
                             "</span>")))))))

(defn- batch-row
  [ledger {:keys [id product quality-grade length-meters shipped-length-meters
                  breaking-strength-kn defect-rate-percent
                  verified? registered?] :as _b}]
  (format (str "        <tr><td><code>%s</code></td><td>%s</td><td>%s</td>"
               "<td>%s</td><td>%s</td><td>%s</td><td>%s</td><td>%s</td>"
               "<td>%s</td><td>%s</td></tr>")
          (esc id) (esc product) (esc (kw-name quality-grade))
          (num->s length-meters) (num->s shipped-length-meters)
          (num->s (when (and (number? length-meters) (number? shipped-length-meters))
                    (- (double length-meters) (double shipped-length-meters))))
          (num->s breaking-strength-kn) (num->s defect-rate-percent)
          (str (flag-cell verified? "verified" "UNVERIFIED") " / "
               (flag-cell registered? "registered" "unregistered"))
          (status-cell ledger id)))

(defn- equipment-row
  [maint {:keys [id kind verified? registered?
                 last-maintenance-date last-scheduled-maintenance-date] :as _e}]
  (format (str "        <tr><td><code>%s</code></td><td>%s</td><td>%s</td>"
               "<td>%s</td><td>%s</td><td>%s</td></tr>")
          (esc id) (esc (kw-name kind))
          (str (flag-cell verified? "verified" "UNVERIFIED") " / "
               (flag-cell registered? "registered" "unregistered"))
          (if last-maintenance-date (esc last-maintenance-date)
              "<span class=\"muted\">never</span>")
          (if last-scheduled-maintenance-date
            (str "<span class=\"ok\">" (esc last-scheduled-maintenance-date) "</span>")
            "<span class=\"muted\">none this run</span>")
          (count (filter #(= id (:equipment-id %)) maint))))

(defn- draft-row
  "One committed registry DRAFT (`cordageops.registry/register-maintenance`
  / `register-shipment` output, string-keyed by construction)."
  [r]
  (format "        <tr><td><code>%s</code></td><td>%s</td><td><code>%s</code></td><td>%s</td></tr>"
          (esc (get r "record_id")) (esc (get r "kind"))
          (esc (or (get r "maintenance_id") (get r "shipment_id")))
          (esc (or (get r "equipment_id") "—"))))

(defn- concern-row [{:keys [id equipment-id concern-type severity description]}]
  (format "        <tr><td><code>%s</code></td><td><code>%s</code></td><td>%s</td><td>%s</td><td>%s</td></tr>"
          (esc id) (esc equipment-id) (esc (kw-name concern-type))
          (esc (kw-name severity)) (esc description)))

(defn- ledger-row [{:keys [t op subject basis violations summary]}]
  (format (str "        <tr><td>%s</td><td><code>%s</code></td><td><code>%s</code></td>"
               "<td>%s</td><td>%s</td></tr>")
          (if (= :committed t)
            "<span class=\"ok\">committed</span>"
            (str "<span class=\"critical\">" (esc (kw-name t)) "</span>"))
          (esc (kw-name op)) (esc subject)
          (esc (str/join ", " (map kw-name basis)))
          (if (seq violations)
            (esc (str/join " / " (map :detail violations)))
            (esc summary))))

(def ^:private action-gate-rows
  ;; Hand-written: this documents the actor's FIXED op contract
  ;; (`cordageops.governor/allowed-ops`, `cordageops.phase/phases`),
  ;; not a runtime measurement. Labelled as such on the page.
  ["        <tr><td><code>:log-production-batch</code></td><td><span class=\"ok\">phase-3 auto-commit when governor-clean</span> &middot; grade / breaking-strength / defect-rate validated against the closed plausible ranges</td></tr>"
   "        <tr><td><code>:schedule-maintenance</code></td><td><span class=\"warn\">ALWAYS human approval</span> &middot; never in any phase's <code>:auto</code> set &middot; equipment <code>:verified?</code> AND <code>:registered?</code> re-derived independently &middot; <code>:direct-operate? true</code> permanently blocked</td></tr>"
   "        <tr><td><code>:flag-safety-concern</code></td><td><span class=\"warn\">ALWAYS human approval</span> &middot; always <code>:stake :coordination/safety-concern</code> &middot; never gated on the equipment being verified (safety reporting is never blocked on paperwork)</td></tr>"
   "        <tr><td><code>:coordinate-shipment</code></td><td><span class=\"warn\">ALWAYS human approval</span> &middot; batch <code>:verified?</code> AND <code>:registered?</code> re-derived independently &middot; shipped-length headroom recomputed from the batch's own record, never from the proposal's claim</td></tr>"
   "        <tr><td><em>anything else</em></td><td><span class=\"critical\">HARD hold</span> &middot; the op allowlist and the proposal-effect allowlist are both closed &middot; a request whose own <code>:effect</code> is not <code>:propose</code> is rejected before any other check</td></tr>"])

(defn render
  "Renders the whole operator-console document from a store `db` that has
  already been driven by `run-demo!` (or any other real scenario)."
  [db]
  (let [ledger (vec (store/ledger db))
        batches (store/all-batches db)
        equipment (store/all-equipment db)
        maint (store/all-maintenance db)
        drafts (concat (store/maintenance-history db) (store/shipment-history db))
        concerns (store/safety-concerns db)
        holds (filter #(= :governor-hold (:t %)) ledger)
        commits (filter #(= :committed (:t %)) ledger)]
    (str
     "<!doctype html>\n"
     "<html lang=\"en\"><head><meta charset=\"utf-8\">"
     "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">"
     "<title>cloud-itonami-isic-1394 &middot; cordage, rope, twine and netting</title><style>"
     (jp-go-dds.skin/dds+skin)
     "</style></head><body>\n"
     "<header class=\"bar\">\n"
     "  <h1>Manufacture of cordage, rope, twine and netting (ISIC 1394) — Operator Console</h1>\n"
     "  <span class=\"badge\">read-only sample &middot; governor-gated &middot; maintenance / safety / shipment always human-approved</span>\n"
     "</header>\n"
     "<main>\n"

     "  <section class=\"card\">\n"
     "    <h2>This run</h2>\n"
     "    <p class=\"muted\">Build-time generated by <code>cordageops.render-html</code> (<code>clojure -M:dev:render-html</code>) by executing the real <code>cordageops.operation</code> StateGraph over a freshly seeded <code>cordageops.store</code>. Every id, number and disposition below is that run's actual output — nothing on this page is hand-typed except the action-gate table, which documents the fixed op contract.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Ledger facts</th><th>Committed</th><th>HARD holds</th><th>Maintenance drafts</th><th>Shipment drafts</th><th>Safety concerns</th></tr></thead>\n"
     "      <tbody>\n"
     (format "        <tr><td>%s</td><td><span class=\"ok\">%s</span></td><td><span class=\"critical\">%s</span></td><td>%s</td><td>%s</td><td>%s</td></tr>"
             (count ledger) (count commits) (count holds)
             (count (store/maintenance-history db))
             (count (store/shipment-history db))
             (count concerns))
     "\n      </tbody>\n"
     "    </table>\n"
     "  </section>\n"

     "  <section class=\"card\">\n"
     "    <h2>Production batches</h2>\n"
     "    <p class=\"muted\">SSoT state after the run. <em>Remaining</em> is the batch's own recorded production length minus its own cumulative shipped length — the same ground truth <code>cordageops.registry/shipment-volume-exceeded?</code> recomputes, never a self-reported figure.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Batch</th><th>Product</th><th>Grade</th><th>Length (m)</th><th>Shipped (m)</th><th>Remaining (m)</th><th>Breaking strength (kN)</th><th>Defect rate (%)</th><th>Ground truth</th><th>Ledger for this subject</th></tr></thead>\n"
     "      <tbody>\n"
     (str/join "\n" (map (partial batch-row ledger) batches)) "\n"
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"

     "  <section class=\"card\">\n"
     "    <h2>Equipment</h2>\n"
     "    <p class=\"muted\">Maintenance may only ever be scheduled against a unit that is independently <code>:verified?</code> AND <code>:registered?</code>. Directly operating a fibre-twisting machine, braiding machine or winding line is permanently blocked — this actor drafts maintenance windows, it never runs a line.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Unit</th><th>Kind</th><th>Ground truth</th><th>Last maintenance</th><th>Scheduled this run</th><th>Windows on file</th></tr></thead>\n"
     "      <tbody>\n"
     (str/join "\n" (map (partial equipment-row maint) equipment)) "\n"
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"

     "  <section class=\"card\">\n"
     "    <h2>Committed drafts</h2>\n"
     "    <p class=\"muted\">Unsigned DRAFT records produced by <code>cordageops.registry</code>. Record numbers come from the store's own sequence counters. A draft is a record a plant coordinator keeps — it never actuates equipment and never dispatches a freight carrier.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Record</th><th>Kind</th><th>Subject</th><th>Equipment</th></tr></thead>\n"
     "      <tbody>\n"
     (if (seq drafts)
       (str (str/join "\n" (map draft-row drafts)) "\n")
       "        <tr><td colspan=\"4\"><span class=\"muted\">no drafts committed in this run</span></td></tr>\n")
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"

     "  <section class=\"card\">\n"
     "    <h2>Safety-concern log</h2>\n"
     "    <p class=\"muted\">Append-only. A safety concern always escalates to a human plant supervisor regardless of confidence, and is never blocked on the referenced equipment being verified.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Concern</th><th>Equipment</th><th>Type</th><th>Severity</th><th>Description</th></tr></thead>\n"
     "      <tbody>\n"
     (if (seq concerns)
       (str (str/join "\n" (map concern-row concerns)) "\n")
       "        <tr><td colspan=\"5\"><span class=\"muted\">no concerns flagged in this run</span></td></tr>\n")
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"

     "  <section class=\"card\">\n"
     "    <h2>Action gate (Cordage &amp; Netting Plant Operations Governor)</h2>\n"
     "    <p class=\"muted\">Fixed contract — hand-written documentation of <code>cordageops.governor/allowed-ops</code> and <code>cordageops.phase/phases</code>, not runtime telemetry. HARD holds cannot be overridden by any phase or any approver.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Op</th><th>Gate</th></tr></thead>\n"
     "      <tbody>\n"
     (str/join "\n" action-gate-rows) "\n"
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"

     "  <section class=\"card\">\n"
     "    <h2>Audit ledger (this run)</h2>\n"
     "    <p class=\"muted\">The append-only decision log the <code>:commit</code> and <code>:hold</code> nodes actually wrote. Every HARD-hold row below was rejected at <code>:decide</code> and routed straight to <code>:hold</code> — no human was ever asked.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Fact</th><th>Op</th><th>Subject</th><th>Basis</th><th>Governor detail / commit summary</th></tr></thead>\n"
     "      <tbody>\n"
     (str/join "\n" (map ledger-row ledger)) "\n"
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"

     "</main>\n"
     "</body></html>\n")))

(defn -main [& args]
  (let [out (or (first args) "docs/samples/operator-console.html")
        db (run-demo!)
        html (render db)
        ledger (store/ledger db)]
    (io! (let [f (java.io.File. ^String out)]
           (when-let [p (.getParentFile f)] (.mkdirs p))
           (spit f html)))
    (println "wrote" out
             (str "(" (count ledger) " ledger facts, "
                  (count (filter #(= :governor-hold (:t %)) ledger)) " HARD holds, "
                  (count (store/maintenance-history db)) " maintenance drafts, "
                  (count (store/shipment-history db)) " shipment drafts, "
                  (count (store/safety-concerns db)) " safety concerns)"))))
