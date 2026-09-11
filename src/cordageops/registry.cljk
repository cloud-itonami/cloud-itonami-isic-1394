(ns cordageops.registry
  "Pure-function domain logic for the cordage/rope/twine/netting-plant
  operations coordination actor -- equipment/batch verification,
  shipment-length recompute, quality-grade validation, breaking-
  strength-test plausibility validation, defect-rate plausibility
  validation, and draft maintenance-schedule/shipment-coordination
  record construction.

  This vertical has NO pre-existing `kotoba-lang/cordage`-style
  capability library to wrap (verified: no such repo exists in this
  workspace). The domain logic therefore lives here as pure functions,
  re-verified INDEPENDENTLY by `cordageops.governor` -- the same
  'ground truth, not self-report' discipline every sibling actor's own
  registry establishes (most directly `knittingops.registry`'s
  `shipment-volume-exceeded?`, this build's own closest domain
  analog): never trust a proposal's own self-reported shipped-length/
  status when the inputs needed to recompute it independently are
  already on record.

  This namespace is pure data + pure functions -- no I/O, no network
  call to any real plant-management system. It builds the DRAFT record
  a plant coordinator would keep (a scheduled maintenance window, a
  coordinated shipment), not the act of actuating a fibre-twisting
  machine, braiding machine, or winding line, or dispatching a real
  freight carrier (this actor NEVER does either -- see README `What
  this actor does NOT do`).")

;; ----------------------------- constants -----------------------------

(def valid-grades
  "The closed set of cordage/rope/twine/netting batch quality-grade
  values a production-batch record may declare (post-inspection QC
  classification, informed by the batch's own breaking-strength-test
  result). Anything else is a fabricated/unrecognized grade -- the
  governor HARD-holds rather than let an invented grade pass through."
  #{:grade-a :grade-b :grade-c :substandard :reject})

(def defect-rate-min-percent
  "Physical floor for a batch's own logged defect-rate reading (a
  batch cannot have a negative fraction of defective length -- broken
  strands, missed knots, out-of-tolerance lay length)."
  0.0)

(def defect-rate-max-percent
  "Physical ceiling for a batch's own logged defect-rate reading (at
  most all length in the batch is defective). A reading above this is
  implausible inspection/scale data, not a real batch."
  100.0)

(def breaking-strength-min-kn
  "Physical floor for a batch's own logged breaking-strength-test
  reading, in kilonewtons. A rope/twine/netting sample that did not
  break under any tension is not a valid breaking-strength reading --
  the value must be a strictly positive load."
  0.0)

(def breaking-strength-max-kn
  "Physical ceiling for a batch's own logged breaking-strength-test
  reading, in kilonewtons -- generous enough to cover the full product
  range this vertical manufactures (light twine through heavy
  synthetic-fibre marine mooring rope), but a reading beyond this is
  implausible instrument/transcription error, not a real breaking-
  strength-test result."
  20000.0)

;; ----------------------------- equipment checks -----------------------------

(defn equipment-verified?
  "Ground-truth check: has `equipment`'s own record been marked
  verified (i.e. it has actually been inspected/commissioned and
  registered in the SSoT, not merely referenced from an unverified
  maintenance request)? A pure predicate over the equipment's own
  permanent field -- no proposal inspection needed."
  [equipment]
  (true? (:verified? equipment)))

(defn equipment-registered?
  "Ground-truth check: does `equipment`'s own record carry a
  `:registered?` true flag (i.e. it is on file in the plant's
  equipment registry)? Scheduling maintenance against equipment that
  is not on file and registered is the exact scope violation this
  actor's HARD invariant ('plant/batch record must be independently
  verified/registered before any action') exists to block."
  [equipment]
  (true? (:registered? equipment)))

(defn equipment-ready?
  "Combined ground-truth gate: the equipment must be both `verified?`
  AND `registered?` before ANY maintenance may be scheduled against
  it. Two independent facts on the equipment's own permanent record,
  neither inferred from the advisor's own rationale."
  [equipment]
  (and (equipment-verified? equipment) (equipment-registered? equipment)))

;; ----------------------------- batch checks -----------------------------

(defn batch-verified?
  "Ground-truth check: has `batch`'s own record been marked verified
  (i.e. its quality-grade/length/breaking-strength/defect-rate claims
  have actually been QC-inspected, not merely logged from an
  unverified intake patch)?"
  [batch]
  (true? (:verified? batch)))

(defn batch-registered?
  "Ground-truth check: is `batch`'s own record on file in the plant's
  production ledger? Coordinating a shipment against a batch that is
  not on file and registered is the exact scope violation this
  actor's HARD invariant ('plant/batch record must be independently
  verified/registered before any action') exists to block."
  [batch]
  (true? (:registered? batch)))

(defn batch-ready?
  "Combined ground-truth gate: the batch must be both `verified?` AND
  `registered?` before ANY shipment may be coordinated against it."
  [batch]
  (and (batch-verified? batch) (batch-registered? batch)))

(defn shipment-volume-exceeded?
  "Ground-truth check for a `:coordinate-shipment` proposal: would
  `shipped-to-date-meters` + `new-length-meters` exceed `batch`'s own
  recorded `:length-meters` (the batch's own logged production
  length)? Needs no proposal inspection or stored-verdict lookup --
  its inputs are permanent fields already on the batch's own record,
  the same shape every sibling actor's own cost/total-matching check
  uses (most directly `knittingops.registry/shipment-volume-exceeded?`,
  this build's own closest domain analog)."
  [batch new-length-meters]
  (let [capacity (:length-meters batch)
        so-far (:shipped-length-meters batch 0.0)]
    (and (number? capacity)
         (number? new-length-meters)
         (number? so-far)
         ;; Compared at 1/10000 of a unit, not on raw doubles. A shipment
         ;; that fills a batch EXACTLY to its recorded capacity is legal,
         ;; and comparing the raw sum flagged such shipments as over
         ;; because the sum is not the double nearest the true total.
         (> (Math/round (* 10000 (+ (double so-far) (double new-length-meters))))
            (Math/round (* 10000 (double capacity))))))) 

(defn shipment-volume-exceeded-checkable?
  "Can `batch`'s headroom actually be computed for `new-length-meters`?

  `shipment-volume-exceeded?` answers only `over` / `not over`, and its
  `(and (number? ...) ...)` guard made every un-checkable case fall
  through as `not over` -- a batch with no recorded capacity, or a
  shipment stating no amount, passed the over-capacity check silently.
  Callers must ask this first: un-checkable is not headroom."
  [batch new-length-meters]
  (boolean (and (map? batch)
                (number? (:length-meters batch))
                (number? (:shipped-length-meters batch 0.0))
                (number? new-length-meters))))

(defn grade-valid?
  "Is `grade` one of the closed, known quality-grade values? nil/blank
  is treated as invalid (a production-batch patch must declare a real
  grade, not omit it silently)."
  [grade]
  (contains? valid-grades grade))

(defn defect-rate-valid?
  "Is `percent` a physically plausible batch-level defect-rate
  reading? Rejects nil, non-numbers, negative values, and values
  beyond `defect-rate-max-percent` -- a fabricated or
  inspection-error reading, never let through as a real batch fact."
  [percent]
  (and (number? percent)
       (>= (double percent) defect-rate-min-percent)
       (<= (double percent) defect-rate-max-percent)))

(defn breaking-strength-valid?
  "Is `kn` a physically plausible batch-level breaking-strength-test
  reading (in kilonewtons)? Rejects nil, non-numbers, non-positive
  values (a rope/twine/netting sample must actually break under some
  positive load for the reading to be a real breaking-strength-test
  result), and values beyond `breaking-strength-max-kn` -- a
  fabricated or instrument-error reading, never let through as a real
  batch fact."
  [kn]
  (and (number? kn)
       (> (double kn) breaking-strength-min-kn)
       (<= (double kn) breaking-strength-max-kn)))

;; ----------------------------- draft record construction -----------------------------

(defn- unsigned-certificate
  "Every certificate this actor produces is UNSIGNED -- signature is
  the human plant supervisor's/shipping approver's act, not this
  actor's."
  [kind subject record-id]
  {"@context" ["https://www.w3.org/ns/credentials/v2"]
   "type" ["VerifiableCredential" kind]
   "credentialSubject" {"id" subject "record" record-id}
   "proof" nil
   "issued_by_registry" false
   "status" "draft-unsigned"})

(defn- zero-pad [n w]
  (let [s (str n)]
    (str (apply str (repeat (max 0 (- w (count s))) "0")) s)))

(defn register-maintenance
  "Validate + construct the MAINTENANCE-SCHEDULE DRAFT -- a proposed
  fibre-twisting-machine, braiding-machine, or winding-line
  maintenance window against a verified, registered piece of
  equipment. Pure function -- does not actuate twisting/braiding/
  winding-line equipment or execute any maintenance; it builds the
  RECORD a plant coordinator would keep. `cordageops.governor`
  independently re-verifies the equipment's own verified/registered
  ground truth, and permanently blocks any attempt to set
  `:direct-operate? true` on a maintenance proposal (see README `What
  this actor does NOT do`), before this is ever allowed to commit."
  [maintenance-id equipment-id sequence]
  (when-not (and maintenance-id (not= maintenance-id ""))
    (throw (ex-info "maintenance: maintenance_id required" {})))
  (when-not (and equipment-id (not= equipment-id ""))
    (throw (ex-info "maintenance: equipment_id required" {})))
  (when (< sequence 0)
    (throw (ex-info "maintenance: sequence must be >= 0" {})))
  (let [maintenance-number (str "MNT-" (zero-pad sequence 6))
        record {"record_id" maintenance-number
                "kind" "maintenance-schedule-draft"
                "maintenance_id" maintenance-id
                "equipment_id" equipment-id
                "immutable" true}]
    {"record" record "maintenance_number" maintenance-number
     "certificate" (unsigned-certificate "MaintenanceSchedule" maintenance-number maintenance-number)}))

(defn register-shipment
  "Validate + construct the SHIPMENT-COORDINATION DRAFT -- a proposed
  outbound cordage/rope/twine/netting shipment against a verified,
  registered production batch. Pure function -- does not dispatch any
  real freight carrier; it builds the RECORD a plant coordinator would
  keep. `cordageops.governor` independently re-verifies the shipment's
  own claimed length against `shipment-volume-exceeded?`, before this
  is ever allowed to commit."
  [shipment-id sequence]
  (when-not (and shipment-id (not= shipment-id ""))
    (throw (ex-info "shipment: shipment_id required" {})))
  (when (< sequence 0)
    (throw (ex-info "shipment: sequence must be >= 0" {})))
  (let [shipment-number (str "SHP-" (zero-pad sequence 6))
        record {"record_id" shipment-number
                "kind" "shipment-coordination-draft"
                "shipment_id" shipment-id
                "immutable" true}]
    {"record" record "shipment_number" shipment-number
     "certificate" (unsigned-certificate "ShipmentCoordination" shipment-number shipment-number)}))

(defn append [history result]
  (conj (vec history) (get result "record")))
