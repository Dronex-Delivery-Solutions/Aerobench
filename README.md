# Aerobench
diff --git a/README.md b/README.md
index 6233a4978a539cd39c4e77b4c047675bc15adb4c..71cfc127cc3e1c9f5877cb543029eba29ee7a804 100644
--- a/README.md
+++ b/README.md
@@ -1 +1,567 @@
 # Aerobench
+
+Aerobench is a proposed benchmarking and product-development-readiness (PDR) suite for evaluating robotics trajectory-estimation systems used by DroneX Delivery Solutions. It is intended to compare SLAM, LiDAR odometry, visual-inertial odometry (VIO), GNSS/INS fusion, fiducial/marker localization, and hybrid autonomy stacks across repeatable datasets, field trials, and simulation scenarios.
+
+## Inspiration and positioning
+
+Projects such as [evo](https://github.com/MichaelGrupp/evo) and [SLAM-Hive](https://slam-hive.net/) demonstrate the value of repeatable trajectory evaluation, common metrics, containerized algorithm execution, and experiment provenance. Aerobench should build on those lessons while focusing on DroneX delivery constraints:
+
+- aerial and ground delivery vehicle motion profiles;
+- mixed indoor, depot, curbside, sidewalk, road-edge, rooftop, and customer-drop-off environments;
+- GPS-denied, GPS-degraded, reflective, low-texture, dark, rainy, windy, high-dynamic-range, and dynamic-obstacle conditions;
+- safety, latency, reproducibility, fleet-scale regression detection, and product release gates;
+- business-facing scorecards that translate algorithm performance into operational risk.
+
+## Product vision
+
+Aerobench should become the internal source of truth for localization and mapping readiness. A successful version of the platform lets the robotics team answer:
+
+1. Which trajectory algorithm is safest and most reliable for a given vehicle, payload, environment, and mission profile?
+2. What changed between two algorithm builds, and did that change improve field performance or merely optimize one dataset?
+3. What sensor combinations are worth their cost, power, weight, maintenance burden, and compute load?
+4. Which scenarios remain risky enough to block a product release?
+5. How do offline benchmark results correlate with live autonomy interventions, customer delivery success, and vehicle recovery events?
+
+## Core design principles
+
+- **Reproducibility first:** every benchmark run records code revision, container digest, dataset version, calibration files, parameter set, hardware target, random seeds, and metric definitions.
+- **Dataset immutability:** raw logs are content-addressed and never modified in place; derived artifacts are versioned separately.
+- **Metric transparency:** all scores are decomposable into raw metric values, scenario weights, thresholds, and failure annotations.
+- **Operational relevance:** benchmark scenarios should map to delivery risk, not just academic dataset leaderboards.
+- **Comparable execution:** algorithms run behind a standard adapter contract so outputs can be compared even when the source systems differ.
+- **Human-reviewable failures:** every failed run should produce plots, aligned trajectories, event timelines, sensor health summaries, and short natural-language diagnostics.
+- **CI-friendly regression detection:** small smoke datasets run quickly on every relevant pull request; larger suites run nightly or before releases.
+- **Extensible by design:** new datasets, sensors, algorithms, metrics, and report templates should be added without rewriting the orchestration layer.
+
+## Recommended high-level architecture
+
+```text
+                    +-------------------------------+
+                    |        Aerobench Portal       |
+                    | dashboards, reports, reviews  |
+                    +---------------+---------------+
+                                    |
+                                    v
++-------------+      +--------------+--------------+      +----------------+
+| Dataset     | ---> | Benchmark Orchestrator       | ---> | Result Store   |
+| Registry    |      | scheduling, containers, DAGs |      | metrics, plots |
++------+------+      +------+----------------------+      +-------+--------+
+       |                    |                                      |
+       v                    v                                      v
++------+---------+   +------+------------------+          +--------+--------+
+| Object Storage |   | Algorithm Adapters      |          | Analysis APIs   |
+| bags, logs, GT |   | SLAM, LiDAR, VIO, INS   |          | notebooks, SDK  |
++----------------+   +-------------------------+          +-----------------+
+```
+
+### Suggested service boundaries
+
+| Component | Responsibility | Recommended technologies |
+| --- | --- | --- |
+| Dataset registry | Dataset metadata, lineage, splits, license, scenario tags, sensor manifest, calibration links | PostgreSQL, SQLAlchemy/Alembic, Pydantic, object-store URIs |
+| Artifact store | Raw logs, normalized logs, ground truth, generated trajectories, plots, reports | S3-compatible storage, content hashes, lifecycle policies |
+| Benchmark orchestrator | Run planning, queueing, retries, dependency DAGs, resource allocation, provenance capture | Python, FastAPI, Celery/RQ/Arq, Kubernetes Jobs, Docker/OCI images |
+| Algorithm adapter SDK | Standard input/output contracts for each algorithm family | Python package, ROS 2 launch wrappers, protobuf/JSON schemas |
+| Metrics engine | Alignment, ATE/RPE, drift, latency, robustness, failure classification, score aggregation | Python, NumPy, SciPy, evo integration, pandas/Polars |
+| Report generator | Plots, tables, scenario summaries, release-gate evidence | Plotly, Matplotlib, Jinja2, static HTML/PDF exports |
+| Web portal | Experiment creation, dashboards, comparison views, reviewer workflow | React/Next.js or lightweight FastAPI templates depending on team size |
+| Experiment database | Run status, metrics, comparisons, annotations, release decisions | PostgreSQL, TimescaleDB extension if time-series queries dominate |
+| Observability | Logs, traces, queue metrics, resource usage, benchmark health | OpenTelemetry, Prometheus, Grafana, structured JSON logs |
+
+## PDR modules
+
+The following PDR modules are intended to turn Aerobench from a benchmark script collection into a release-quality engineering platform. Each module includes objective, core features, outputs, and readiness checks.
+
+### PDR-01: Program charter and benchmark taxonomy
+
+**Objective:** Define what Aerobench evaluates, why it matters, and how results influence product decisions.
+
+**Core features**
+
+- Algorithm taxonomy: visual SLAM, LiDAR SLAM, VIO, wheel/leg odometry, GNSS/INS, fiducial localization, map-based localization, multi-sensor fusion.
+- Vehicle taxonomy: drone, delivery rover, test cart, simulation agent, lab rig.
+- Environment taxonomy: warehouse, sidewalk, road edge, parking lot, elevator/lobby, rooftop, depot yard, suburban neighborhood.
+- Scenario taxonomy: low texture, dynamic agents, GNSS multipath, rain/fog, lighting transitions, vibration, payload shift, aggressive turns, loop closures, feature-poor corridors, reflective glass.
+- Decision taxonomy: research comparison, sensor selection, regression detection, release gate, incident reproduction, procurement support.
+
+**Outputs**
+
+- Benchmark charter.
+- Scenario naming convention.
+- Release-gate policy draft.
+- Baseline algorithm list.
+
+**Readiness checks**
+
+- Every benchmark scenario maps to at least one operational risk.
+- Every scorecard metric has an owner and a product decision it supports.
+- Leadership, robotics, safety, and operations agree on blocking vs. advisory metrics.
+
+### PDR-02: Dataset registry and scenario library
+
+**Objective:** Make every dataset discoverable, versioned, auditable, and safely reusable.
+
+**Core features**
+
+- Immutable raw dataset IDs based on content hashes.
+- Metadata for date, site, vehicle, sensors, firmware, weather, lighting, route, operator, and known anomalies.
+- Scenario tags with controlled vocabulary and free-text notes.
+- Dataset splits: smoke, development, validation, release, stress, incident, and hidden holdout.
+- Ground-truth provenance: motion-capture, survey-grade GNSS, total station, AprilTag grid, LiDAR map alignment, simulation truth, or hand-audited reference.
+- Calibration and synchronization manifests.
+
+**Outputs**
+
+- `datasets.yaml` or database-backed registry.
+- Scenario catalogue.
+- Dataset quality report.
+- Data-retention and access-control policy.
+
+**Readiness checks**
+
+- No benchmark can run against an unversioned dataset.
+- Dataset schema validates in CI.
+- Sensitive location/customer information is redacted or access-controlled.
+- Ground-truth uncertainty is recorded, not hidden.
+
+### PDR-03: Algorithm adapter contract
+
+**Objective:** Let different robotics stacks run through one repeatable interface.
+
+**Core features**
+
+- Standard input contract: dataset URI, sensor topics, calibration bundle, time window, parameter file, output directory, resource limits.
+- Standard output contract: trajectory, covariance when available, map artifact, status code, runtime logs, resource metrics, warnings, diagnostics.
+- Supported trajectory formats: TUM, KITTI, EuRoC, ROS bag/rosbag2 topics, CSV, and internal protobuf/Parquet representation.
+- Adapter modes: offline replay, real-time playback, simulation, hardware-in-the-loop, and live shadow mode.
+- Failure semantics: initialization failure, tracking loss, map corruption, missing output, timeout, resource exhaustion, invalid timestamps, unstable covariance.
+
+**Outputs**
+
+- Adapter SDK.
+- Example adapters for one SLAM, one LiDAR odometry, one VIO, and one baseline dead-reckoning algorithm.
+- Contract tests with synthetic mini-datasets.
+
+**Readiness checks**
+
+- A new adapter can be implemented without changing the metrics engine.
+- Bad outputs fail validation before metrics are computed.
+- Adapter logs include enough detail for reproducibility and debugging.
+
+### PDR-04: Metrics and scoring engine
+
+**Objective:** Convert trajectories and run artifacts into trustworthy, explainable metrics.
+
+**Core metric families**
+
+- Accuracy: absolute trajectory error (ATE), relative pose error (RPE), segment drift, endpoint error, heading error, altitude error, scale error.
+- Robustness: tracking-loss count, relocalization time, loop-closure consistency, catastrophic divergence, invalid pose gaps, failure-to-initialize rate.
+- Timing: end-to-end latency, pose publication frequency, jitter, startup time, relocalization latency.
+- Resource usage: CPU, GPU, memory, disk I/O, network I/O, power proxy, thermal throttling events.
+- Consistency: covariance calibration, normalized estimation error squared where ground truth and covariance are available.
+- Map quality: map completeness, occupancy consistency, point-cloud density, semantic stability, loop-closure deformation, map update latency.
+- Operational safety: geofence violation, obstacle-proximity localization error, pose jumps during control-critical windows, autonomy disengagement correlation.
+
+**Scoring guidance**
+
+- Keep raw metrics and product scores separate.
+- Use scenario-specific thresholds rather than a single global score.
+- Report confidence intervals across repeated runs and bootstrapped route segments.
+- Publish both pass/fail gates and continuous scores.
+- Penalize missing or invalid outputs explicitly.
+- Record alignment choices because alignment can hide real product risks.
+
+**Outputs**
+
+- Metric schema.
+- Scoring configuration files.
+- Comparison reports.
+- Regression detector.
+
+**Readiness checks**
+
+- Metric formulas are documented and unit-tested.
+- Score changes can be traced back to raw metric changes.
+- Release gates are deterministic for the same inputs.
+
+### PDR-05: Ground truth, calibration, and time synchronization
+
+**Objective:** Reduce false conclusions caused by poor references, bad calibration, or clock drift.
+
+**Core features**
+
+- Ground-truth quality classes with uncertainty bounds.
+- Calibration bundle versioning for intrinsics, extrinsics, IMU noise, wheel parameters, LiDAR timing, camera rolling-shutter parameters, GNSS antenna offsets.
+- Clock synchronization checks across sensors.
+- Time-offset estimation diagnostics.
+- Ground-truth alignment policy by benchmark type.
+- Calibration drift monitoring over vehicle lifetime.
+
+**Outputs**
+
+- Calibration manifest schema.
+- Ground-truth certification checklist.
+- Time-sync validation tool.
+- Dataset rejection or quarantine workflow.
+
+**Readiness checks**
+
+- Benchmark reports show reference quality level.
+- Time jumps and unsynchronized streams are detected automatically.
+- Calibration used for a run is immutable and linked to the result.
+
+### PDR-06: Orchestration and reproducible execution
+
+**Objective:** Run large experiment matrices reliably without losing provenance.
+
+**Core features**
+
+- Containerized algorithm execution with pinned images.
+- Matrix runs across algorithm versions, parameter sets, datasets, and hardware profiles.
+- Queue priorities for smoke, nightly, release, and incident-reproduction runs.
+- Resource quotas and timeout policies.
+- Retry policy that distinguishes infrastructure failures from algorithm failures.
+- Run manifests stored before execution starts.
+- Deterministic re-run command for every completed experiment.
+
+**Outputs**
+
+- CLI: `aerobench run`, `aerobench compare`, `aerobench report`, `aerobench validate-dataset`.
+- Job runner.
+- Run manifest schema.
+- Provenance capture middleware.
+
+**Readiness checks**
+
+- The same run manifest can be re-executed months later.
+- Partial failures do not corrupt result records.
+- Queue status and worker logs are visible to developers.
+
+### PDR-07: Analysis, visualization, and review workflow
+
+**Objective:** Help engineers understand why an algorithm won or failed.
+
+**Core features**
+
+- Aligned 2D/3D trajectory overlays.
+- Error-over-time and error-over-distance plots.
+- Per-segment heatmaps on route maps.
+- Sensor availability timeline.
+- Tracking state timeline.
+- Resource usage timeline.
+- Side-by-side comparison of two or more runs.
+- Automatic anomaly summaries.
+- Reviewer comments and sign-off records.
+
+**Outputs**
+
+- Static report artifact per run.
+- Interactive comparison dashboard.
+- Release-candidate scorecard.
+- Failure triage board integration.
+
+**Readiness checks**
+
+- A reviewer can reproduce the exact plots from stored artifacts.
+- Reports distinguish data problems from algorithm problems.
+- Every release-blocking failure can be linked to evidence.
+
+### PDR-08: Simulation, synthetic perturbation, and fault injection
+
+**Objective:** Evaluate rare or dangerous conditions before they occur in the field.
+
+**Core features**
+
+- Simulation dataset support from Gazebo/Ignition, AirSim, Unity, Isaac Sim, or internal simulators.
+- Synthetic perturbations: timestamp jitter, dropped frames, motion blur, LiDAR dropout, IMU bias, camera exposure shifts, GNSS multipath, wheel slip, vibration, packet loss.
+- Scenario randomization with seed capture.
+- Domain-gap annotations between simulated and real datasets.
+- Stress-test generator for edge-case sweeps.
+
+**Outputs**
+
+- Perturbation library.
+- Simulation adapter.
+- Synthetic scenario manifest.
+- Robustness stress report.
+
+**Readiness checks**
+
+- Perturbations are parameterized and reproducible.
+- Simulation results are not mixed with real-world release gates without explicit labels.
+- Fault injection has safety review for any hardware-in-the-loop usage.
+
+### PDR-09: Continuous benchmarking and regression gates
+
+**Objective:** Catch localization regressions before they reach vehicles.
+
+**Core features**
+
+- Pull-request smoke suite with small datasets.
+- Nightly benchmark matrix for main branches.
+- Weekly full validation suite.
+- Release-candidate certification run.
+- Baseline comparison against last approved production version.
+- Statistical regression detection with minimum practical effect sizes.
+- Flaky-run detection and quarantine.
+
+**Outputs**
+
+- CI integration.
+- Benchmark status badges.
+- Regression reports.
+- Release gate checklist.
+
+**Readiness checks**
+
+- CI failure messages point to specific scenarios and metrics.
+- Benchmark runtime fits the intended cadence.
+- Release gates can be waived only with traceable approval.
+
+### PDR-10: Safety, compliance, privacy, and governance
+
+**Objective:** Ensure benchmark evidence is usable for internal safety cases and external audits.
+
+**Core features**
+
+- Dataset access control and audit logs.
+- Customer/site privacy redaction process.
+- Safety-critical metric ownership.
+- Incident dataset handling policy.
+- Model/algorithm approval workflow.
+- Retention rules for raw logs and derived artifacts.
+- Exportable evidence packages for safety review.
+
+**Outputs**
+
+- Governance policy.
+- Data classification labels.
+- Audit export.
+- Release sign-off template.
+
+**Readiness checks**
+
+- Sensitive data cannot be accidentally published in public reports.
+- Safety reviewers can trace a release decision back to benchmark evidence.
+- Incident datasets are separated from general development datasets.
+
+### PDR-11: Developer experience and extensibility
+
+**Objective:** Make Aerobench easy for robotics engineers to use without becoming platform experts.
+
+**Core features**
+
+- Cookiecutter/scaffold for new adapters.
+- Local mini-suite for development.
+- Clear error messages for invalid trajectories and schemas.
+- Notebook and Python SDK examples.
+- Documentation for adding datasets, metrics, algorithms, and reports.
+- Typed configuration and schema validation.
+
+**Outputs**
+
+- Developer guide.
+- Adapter template.
+- Example notebooks.
+- Troubleshooting guide.
+
+**Readiness checks**
+
+- A new engineer can run the local sample benchmark in under 30 minutes.
+- Common adapter mistakes are caught by tests.
+- Documentation includes realistic examples, not only API references.
+
+### PDR-12: Product analytics bridge
+
+**Objective:** Connect benchmark outcomes with delivery-business outcomes.
+
+**Core features**
+
+- Mapping from technical metrics to operational KPIs: successful delivery, intervention rate, route completion, customer wait time, vehicle recovery, maintenance cost.
+- Correlation analysis between offline benchmark failures and field autonomy events.
+- Scenario weighting based on route frequency and severity.
+- Fleet-readiness dashboard by city, vehicle type, sensor package, and software release.
+
+**Outputs**
+
+- Operational risk scorecard.
+- Fleet-readiness report.
+- Scenario weighting model.
+- Quarterly benchmark review.
+
+**Readiness checks**
+
+- Product managers can understand score changes without reading robotics logs.
+- Scenario weights are reviewed periodically against real route distributions.
+- Business KPIs never replace raw safety metrics; they complement them.
+
+## Suggested repository roadmap
+
+### Phase 0: Foundation
+
+- Expand this repository from a README into a Python package with `src/aerobench`.
+- Add schemas for datasets, run manifests, algorithm outputs, and metric results.
+- Add a tiny synthetic trajectory dataset for tests.
+- Implement validation commands before implementing expensive orchestration.
+
+### Phase 1: Local benchmark MVP
+
+- Implement a CLI that runs a baseline adapter on local datasets.
+- Integrate evo-compatible trajectory loading and ATE/RPE metrics.
+- Generate a static HTML report with plots and JSON metrics.
+- Add unit tests for schema validation and metric calculations.
+
+### Phase 2: Containerized adapters
+
+- Define Docker image requirements for algorithms.
+- Implement sample adapters for representative SLAM, LiDAR odometry, and VIO systems.
+- Add resource capture and timeout handling.
+- Introduce reproducible run manifests.
+
+### Phase 3: Registry and portal
+
+- Add PostgreSQL-backed dataset and result registries.
+- Build comparison dashboards and release scorecards.
+- Add reviewer annotations and sign-off workflows.
+- Store generated reports in object storage.
+
+### Phase 4: Continuous benchmarking
+
+- Add PR smoke benchmarks.
+- Add nightly benchmark matrices.
+- Add regression detection and baseline comparison.
+- Add release-candidate certification workflows.
+
+### Phase 5: Product-readiness intelligence
+
+- Link benchmark results to field autonomy events.
+- Add scenario weighting from delivery-route analytics.
+- Add fleet-readiness dashboards.
+- Add safety-review evidence exports.
+
+## Best practices for Aerobench's technical stack
+
+### Python backend and metrics
+
+- Use modern Python with strict typing for schemas and metric APIs.
+- Prefer Pydantic or dataclasses for external contracts; avoid passing untyped dictionaries between layers.
+- Keep metric functions pure: input artifacts plus config in, metric results out.
+- Use NumPy/SciPy for geometry-heavy calculations and pandas or Polars for tabular aggregation.
+- Separate coordinate-frame utilities into a well-tested module.
+- Treat timestamp association as a first-class component with explicit tolerance settings.
+- Keep plotting code separate from metric computation so reports cannot change scores.
+- Include golden tests with known trajectories: identical path, constant offset, scale drift, time shift, dropped poses, and loop-closure jump.
+
+### ROS and robotics integration
+
+- Support ROS 2 first for new systems, while providing legacy wrappers if ROS 1 logs are still needed.
+- Normalize ROS bag, rosbag2, TUM, KITTI, EuRoC, CSV, and internal formats into one canonical trajectory representation.
+- Record frame IDs and transform trees with outputs; frame mistakes are common and expensive.
+- Validate monotonic timestamps, duplicate timestamps, unit conventions, quaternion normalization, and coordinate frames before scoring.
+- Keep algorithm launch files inside adapter images, not inside the core benchmark engine.
+- Store calibration and parameters as explicit run inputs; never rely on a developer's local ROS environment.
+
+### Containers and reproducibility
+
+- Pin base images and package versions for benchmark images.
+- Store image digests, not only image tags, in run manifests.
+- Use multi-stage Dockerfiles for smaller runtime images.
+- Run adapters as non-root users where possible.
+- Use resource limits so one algorithm cannot starve the benchmark cluster.
+- Make every generated artifact deterministic or record randomness and seeds.
+- Keep third-party algorithm images separate from Aerobench core images.
+
+### Data engineering
+
+- Use content-addressed storage for raw datasets and generated artifacts.
+- Prefer append-only result records; corrections should create new versions rather than mutate history.
+- Store large logs in object storage and metadata in PostgreSQL.
+- Use Parquet for large derived tabular artifacts.
+- Define dataset quality tiers and exclude low-quality references from release gates.
+- Maintain hidden holdout datasets to reduce overfitting to public/internal benchmark sets.
+- Track dataset coverage by scenario and operational risk.
+
+### API and web portal
+
+- Use FastAPI when the team wants a Python-first stack with strong schema generation.
+- Keep API responses stable and versioned once dashboards or automation depend on them.
+- Use background jobs for long-running benchmark execution; never block web requests on benchmark runs.
+- Use server-side pagination and filtering for large result tables.
+- Build dashboard views around decisions: compare builds, inspect failures, approve releases, and find risky scenarios.
+- Provide downloadable report bundles for offline design reviews.
+
+### Database and schema management
+
+- Use PostgreSQL for core metadata and results.
+- Use migrations from the start with Alembic or equivalent.
+- Store metric definitions and scoring configurations with semantic versions.
+- Add database constraints for uniqueness of dataset IDs, run IDs, artifact hashes, and adapter versions.
+- Avoid storing huge JSON blobs when fields must be queried frequently; promote them to typed columns.
+- Keep raw metric payloads available for audit, even if summary tables are denormalized for dashboards.
+
+### Testing strategy
+
+- Unit-test geometry, alignment, interpolation, timestamp association, and scoring thresholds.
+- Add property-based tests for trajectory transforms and frame conversions.
+- Add contract tests for every adapter.
+- Add integration tests using tiny datasets that finish quickly in CI.
+- Add regression tests for known historical bugs and incident reproductions.
+- Use snapshot tests for report structure, but avoid brittle pixel-perfect plot tests.
+- Run static analysis, formatting, schema validation, and dependency audits in CI.
+
+### Security and privacy
+
+- Strip or protect customer addresses, faces, license plates, Wi-Fi identifiers, and sensitive location metadata.
+- Separate production incident logs from general research datasets.
+- Use least-privilege access to object storage and databases.
+- Scan containers for vulnerabilities before release use.
+- Track third-party algorithm licenses, especially if benchmark results will be shared externally.
+- Sign or checksum critical artifacts used in release decisions.
+
+### Performance and scalability
+
+- Start with a simple local runner, but design manifests so they can move to Kubernetes Jobs later.
+- Cache normalized datasets and repeated preprocessing steps.
+- Parallelize across datasets and algorithms, not within the metric engine first.
+- Collect CPU, memory, GPU, and I/O metrics for every run.
+- Keep smoke suites under a few minutes; full suites can run asynchronously.
+- Profile timestamp association and trajectory interpolation on long logs.
+
+### Documentation and governance
+
+- Document metric formulas and alignment policies near the code that implements them.
+- Require a short benchmark design review when adding a new release-gate metric.
+- Maintain a changelog for scoring changes because score definitions affect historical comparisons.
+- Provide runbooks for failed jobs, bad datasets, suspicious improvements, and release-blocking regressions.
+- Distinguish research dashboards from certification dashboards.
+- Review scenario weights with operations and safety teams on a fixed cadence.
+
+## Common pitfalls to avoid
+
+- Optimizing for a single public dataset and then assuming field readiness.
+- Reporting only aggregate scores that hide catastrophic failures in rare scenarios.
+- Allowing alignment choices to mask scale drift, heading drift, or global-frame failures.
+- Mixing datasets with different ground-truth quality without labels.
+- Comparing algorithms with different sensor inputs without an explicit sensor-cost scorecard.
+- Ignoring latency and resource usage until late in product development.
+- Letting teams manually edit result artifacts after a run.
+- Treating simulation as a substitute for field validation rather than a complement.
+- Failing to version scoring logic, thresholds, and calibration files.
+- Making the benchmark so heavyweight that engineers avoid running it.
+
+## Near-term implementation checklist
+
+- [ ] Define dataset, run, adapter-output, metric-result, and report schemas.
+- [ ] Add a synthetic trajectory fixture and golden metric tests.
+- [ ] Implement `aerobench validate-dataset`.
+- [ ] Implement `aerobench run-local` for a baseline adapter.
+- [ ] Implement ATE, RPE, drift, latency, and failure-rate metrics.
+- [ ] Generate JSON and static HTML reports.
+- [ ] Add Docker adapter contract tests.
+- [ ] Add CI smoke benchmark.
+- [ ] Add dataset registry backed by a YAML file, then migrate to PostgreSQL when scale requires it.
+- [ ] Add release scorecard template and reviewer sign-off process.
+
+## Definition of done for a first useful release
+
+Aerobench v0.1 is useful when it can run at least three representative algorithms across at least five curated datasets, produce deterministic reports, compare against a baseline, and clearly identify at least one known algorithm weakness that matters to DroneX delivery operations.

