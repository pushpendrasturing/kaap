# Proposal: Debugging/Triage Exercise Bug Candidates

## Repo map (high-level)
- operator/ (Quarkus operator runtime)
  - controllers/:
    - AbstractController + AbstractResourceSetsController: shared reconciliation logic, validation, status/conditions, last-applied diffing.
    - Component controllers: PulsarClusterController (orchestrates all components), BrokerController, BookKeeperController, ProxyController, ZooKeeperController, FunctionsWorkerController, BastionController, AutorecoveryController.
  - controllers/*ResourcesFactory:
    - BaseResourcesFactory builds common Kubernetes objects (labels/annotations, TLS/auth wiring, URLs, readiness helpers).
    - Component factories (Broker/BookKeeper/Proxy/ZooKeeper/etc.) generate configmaps, services, statefulsets/deployments, jobs.
  - autoscaler/:
    - AutoscalerDaemon and NamespacedDaemonThread schedule background autoscaling tasks.
    - Broker/BookKeeper set autoscalers plus resource usage sources (load report, k8s metrics).
    - Bookie admin clients + decommission utilities (exec into pods, read stats).
  - controllers/bookkeeper/racks:
    - BookKeeperRackDaemon/Monitor maintain rack awareness in ZK.
  - controllers/utils:
    - TokenAuthProvisioner (JWT secrets) and CertManagerCertificatesProvisioner (TLS certs).
  - crds/:
    - CRD models + defaulting/validation (GlobalSpec, component specs), SpecDiffer, ConfigUtil.
- operator-common/:
  - SerializationUtil and JSON comparator utilities used across modules.
- migration-tool/ (CLI for migration/diff):
  - Main, SpecGenerator, DiffChecker, diff writers; reads cluster, generates CRDs, compares outputs.
- helm/, docs/: packaging and documentation (not core runtime).

## Bug candidates

### B01 - Reschedule unit mismatch in reconcile
- Location: operator/src/main/java/com/datastax/oss/kaap/controllers/AbstractController.java, reconcile() ~112-181
- Core relevance: shared base reconciler for all CRDs; controls reconcile loop.
- Bug type: time/unit mismatch, performance regression
- Proposed change: call rescheduleAfter with TimeUnit.MILLISECONDS even though config is seconds.
- Trigger conditions: any reconciliation with reschedule=true (not-ready or error paths).
- Expected symptom: tight reconcile loop, high API traffic, log spam; components appear stuck.
- Why its hard: only happens when components are not ready; easy to blame downstream readiness.
- Static-analysis discoverability: Medium (units mismatch is subtle).
- Suggested detection: integration test asserting minimum reschedule interval; monitor reconcile rate.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 5/5

### B02 - Drop newly introduced conditions during merge
- Location: AbstractController.mergeConditions(...) ~296-336
- Core relevance: status/conditions for every controller.
- Bug type: correctness (status drift)
- Proposed change: remove the loop that adds new conditions not in previousConditions.
- Trigger conditions: new condition type appears (e.g., Ready added after first reconcile).
- Expected symptom: status never shows new conditions; external systems see stale state.
- Why its hard: no crash; logs still show reconciliation running.
- Static-analysis discoverability: Low to Medium.
- Suggested detection: unit test for condition merge when previous list is empty.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B03 - Validation errors swallowed
- Location: AbstractController.validate(...) ~208-222
- Core relevance: validation gate for all CRDs.
- Bug type: input validation gap
- Proposed change: return null even when violations exist (e.g., after logging).
- Trigger conditions: invalid spec with constraint violations.
- Expected symptom: invalid spec proceeds, later failures in pods or resources.
- Why its hard: errors may show in logs but status still "Ready" or "Initializing".
- Static-analysis discoverability: Low.
- Suggested detection: unit test asserting invalid spec yields Ready=False condition.
- Rankings: Exercise value 4/5; Stealth 3/5; Scorability 4/5

### B04 - Incorrect lastApplied stored for resource sets
- Location: AbstractResourceSetsController.patchResources(...) ~97-155
- Core relevance: resource-set controllers (broker, bookkeeper, proxy) use this for diffing.
- Bug type: correctness / state drift
- Proposed change: store lastAppliedResource.getSets().put(setName, lastAppliedResource.getCommon()) instead of set-specific spec.
- Trigger conditions: multiple sets with set-specific overrides.
- Expected symptom: repeated patches or missed updates for some sets.
- Why its hard: only manifests with multiple sets and partial updates.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with two sets where only one changes.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B05 - Cleanup deletes active sets
- Location: AbstractResourceSetsController.cleanupDeletedSets(...) ~191-204
- Core relevance: controls deletion of broker/bookkeeper/proxy sets.
- Bug type: data integrity / lifecycle
- Proposed change: compute currentSets incorrectly (e.g., pass Set.of() as excludes).
- Trigger conditions: after spec changes when all sets are ready.
- Expected symptom: active StatefulSets/Services deleted unexpectedly.
- Why its hard: looks like user-driven deletion; only after successful readiness.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test updating sets without deletions.
- Rankings: Exercise value 5/5; Stealth 3/5; Scorability 4/5

### B06 - Defaults skipped for default set
- Location: AbstractResourceSetsController.getSetSpecs(...) ~237-252
- Core relevance: defaulting for broker/bookkeeper/proxy set specs.
- Bug type: correctness / defaulting gap
- Proposed change: when sets are empty, return default set using the raw spec without applyDefaultsWithReflection.
- Trigger conditions: cluster uses implicit default set (no explicit sets).
- Expected symptom: missing default values in generated resources; subtle config drift.
- Why its hard: defaults are implicit; missing values are not obvious.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test comparing applied defaults for implicit set.
- Rankings: Exercise value 3/5; Stealth 3/5; Scorability 3/5

### B07 - Map defaults override user values
- Location: operator/src/main/java/com/datastax/oss/kaap/crds/ConfigUtil.java, mergeMaps(...) ~88-97
- Core relevance: used for defaulting config maps and CRD specs.
- Bug type: correctness / configuration precedence
- Proposed change: reverse precedence so parent defaults override child values.
- Trigger conditions: user sets map-based configuration overrides.
- Expected symptom: user configs silently ignored.
- Why its hard: values still present but revert to defaults.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test in ConfigUtilTest (likely too easy unless the direct test is removed).
- Rankings: Exercise value 3/5; Stealth 2/5; Scorability 4/5

### B08 - Null specs treated as equal
- Location: operator/src/main/java/com/datastax/oss/kaap/crds/SpecDiffer.java, generateDiff(...) ~54-81
- Core relevance: diffing drives updates across controllers.
- Bug type: correctness / change detection
- Proposed change: return RESULT_EQUALS when expectedSpec is null.
- Trigger conditions: spec removed or cleared.
- Expected symptom: operator fails to detect removal and does not update resources.
- Why its hard: only appears on spec deletion; no crash.
- Static-analysis discoverability: Medium.
- Suggested detection: SpecDifferTest (likely too easy unless the direct test is removed).
- Rankings: Exercise value 3/5; Stealth 2/5; Scorability 4/5

### B09 - ConfigMap checksum includes resourceVersion
- Location: BaseResourcesFactory.addConfigMapChecksumAnnotation(...) ~691-700
- Core relevance: affects all components when restartOnConfigMapChange is true.
- Bug type: caching/invalidation, performance regression
- Proposed change: compute checksum from full ConfigMap object (including metadata/resourceVersion).
- Trigger conditions: any reconcile that updates configmap metadata.
- Expected symptom: pods restart on every reconcile, perpetual rollouts.
- Why its hard: looks like config changes even when data is stable.
- Static-analysis discoverability: Low.
- Suggested detection: integration test verifying checksum stability with no data change.
- Rankings: Exercise value 5/5; Stealth 4/5; Scorability 4/5

### B10 - Double-prefix config keys
- Location: BaseResourcesFactory.handleConfigPulsarPrefix(...) ~920-935
- Core relevance: configmap generation for multiple components.
- Bug type: API contract drift (env var naming)
- Proposed change: remove the exception for keys starting with PULSAR_ or BOOKIE_.
- Trigger conditions: config entries already prefixed (common in overrides).
- Expected symptom: settings ignored; components run with defaults.
- Why its hard: configmaps look populated but keys are wrong.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test that verifies prefixing behavior on already-prefixed keys.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B11 - PDB support check uses wrong version comparison
- Location: BaseResourcesFactory.isPdbSupported() ~263-269
- Core relevance: PDB creation used by multiple components; impacts availability guarantees.
- Bug type: reliability / API compatibility
- Proposed change: use "||" instead of "&&" or compare minor version strings lexicographically.
- Trigger conditions: Kubernetes versions around 1.21 boundary or nonstandard minors.
- Expected symptom: wrong apiVersion (policy/v1 vs v1beta1) causing create/patch failures.
- Why its hard: depends on cluster version; may only fail in certain environments.
- Static-analysis discoverability: Low.
- Suggested detection: integration test matrix against 1.20/1.21 clusters.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B12 - TLS scheme/port swap in service URLs
- Location: BaseResourcesFactory.getBrokerServiceUrl()/getProxyServiceUrl() ~409-452
- Core relevance: service URLs are injected into config maps used by brokers/proxies/functions.
- Bug type: API contract drift
- Proposed change: swap TLS and plaintext scheme/port (e.g., pulsar+ssl on 6650).
- Trigger conditions: TLS enabled (or mixed TLS/plain).
- Expected symptom: clients fail to connect; TLS handshake errors or timeouts.
- Why its hard: values look plausible; only fails in TLS scenarios.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test that asserts URL/port mapping under TLS on/off.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B13 - ZooKeeper client port selection swapped
- Location: BaseResourcesFactory.getZkServers(...) ~374-379
- Core relevance: ZooKeeper connection string used by brokers/bookies/proxy.
- Bug type: API contract drift / configuration correctness
- Proposed change: swap TLS and non-TLS client port selection.
- Trigger conditions: TLS enabled or disabled for ZooKeeper.
- Expected symptom: components fail to connect to ZooKeeper; TLS handshake/connection errors.
- Why its hard: looks like TLS/cert issues or network flakiness; only in TLS configs.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test validating ZK connection string for TLS on/off.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B14 - Autoscaler guard removed for bookkeeper
- Location: PulsarClusterController.adjustBookKeeperReplicas(...) ~231-252
- Core relevance: same as B13 but for bookkeepers.
- Bug type: reliability / autoscaler interference
- Proposed change: remove autoscaler enabled guard.
- Trigger conditions: bookkeeper autoscaler enabled.
- Expected symptom: autoscaler updates get reverted; scale-up/down stalls or oscillates.
- Why its hard: only manifests with autoscaler; looks like BK autoscaler bug.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test with autoscaler on and replica changes.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B15 - Missing Ready condition treated as ready
- Location: PulsarClusterController.checkReadyOrPatch(...) ~355-395
- Core relevance: parent controller gates component readiness.
- Bug type: correctness / orchestration
- Proposed change: if Ready condition is missing, default to Ready=true.
- Trigger conditions: newly created child CRs before they set status.
- Expected symptom: parent cluster marked ready prematurely; dependent components start too early.
- Why its hard: race-dependent; failures appear downstream.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test that asserts parent stays not-ready until child Ready condition exists.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 3/5

### B16 - Namespace daemon tasks leak or duplicate
- Location: NamespacedDaemonThread.onSpecChange(...) ~48-61
- Core relevance: autoscaler and rack daemons share this scheduling logic.
- Bug type: concurrency / resource leak
- Proposed change: skip namespaceContext.setCurrent(newSpec) or move cancelTasks after scheduling.
- Trigger conditions: repeated spec updates or multi-namespace usage.
- Expected symptom: duplicated scheduled tasks; multiple autoscalers running at once.
- Why its hard: only apparent over time; looks like autoscaler flapping.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test ensuring only one scheduled task per namespace after repeated updates.
- Rankings: Exercise value 5/5; Stealth 4/5; Scorability 4/5

### B17 - Stabilization window reversed
- Location: AutoscalerUtils.isStsReadyToScale(...) ~79-96
- Core relevance: gating for broker and bookkeeper autoscaling.
- Bug type: time logic error
- Proposed change: use Duration.between(now, podStartTime) (reverse order) or invert comparison with maxStartTime.
- Trigger conditions: stabilizationWindowMs > 0.
- Expected symptom: autoscaler scales too early or never scales.
- Why its hard: timing dependent; appears as "autoscaler too aggressive/slow".
- Static-analysis discoverability: Low to Medium.
- Suggested detection: integration test with controlled pod start times.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B18 - Exec watch/streams not closed
- Location: AutoscalerUtils.execInPod(...) ~176-181
- Core relevance: autoscalers and bookie admin actions exec into pods.
- Bug type: resource leak / reliability
- Proposed change: remove the response.whenComplete closeQuietly block.
- Trigger conditions: repeated exec calls (autoscaler loops).
- Expected symptom: leaked connections, eventual hangs or OOM under long runtimes.
- Why its hard: only manifests in long-lived clusters; intermittent.
- Static-analysis discoverability: Low.
- Suggested detection: load test running autoscaler loops for extended time; monitor open connections.
- Rankings: Exercise value 3/5; Stealth 5/5; Scorability 3/5

### B19 - Broker autoscaler schedule unit mismatch
- Location: BrokerAutoscalerDaemon.specChanged(...) ~56-69
- Core relevance: broker autoscaling path.
- Bug type: time/unit mismatch
- Proposed change: scheduleWithFixedDelay(..., TimeUnit.SECONDS) even though periodMs is in ms.
- Trigger conditions: autoscaler enabled with short periods.
- Expected symptom: scaling runs 1000x slower than configured.
- Why its hard: appears as "autoscaler not reacting"; no errors.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test verifying scheduled delay for periodMs.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B20 - Bookkeeper rack daemon schedule unit mismatch
- Location: BookKeeperRackDaemon.specChanged(...) ~62-75
- Core relevance: rack awareness updates for BookKeeper.
- Bug type: time/unit mismatch
- Proposed change: scheduleWithFixedDelay(..., TimeUnit.SECONDS) while period is ms.
- Trigger conditions: auto-rack enabled.
- Expected symptom: rack updates lag by orders of magnitude.
- Why its hard: only visible in long-running clusters.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test verifying scheduling interval.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B21 - Threshold boundary flip in broker autoscaler
- Location: BrokerSetAutoscaler.decideScaleUpOrDown(...) ~165-190
- Core relevance: broker autoscaling decisions.
- Bug type: numeric precision / boundary handling
- Proposed change: use <= and >= comparisons (or cast thresholds to int).
- Trigger conditions: CPU usage near exact thresholds.
- Expected symptom: scale flapping at the boundary or oscillation.
- Why its hard: only at boundary; logs show legitimate values.
- Static-analysis discoverability: Low.
- Suggested detection: load test around threshold values; property-based test on threshold boundaries.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B22 - Scale-up max limit applied incorrectly
- Location: BookKeeperSetAutoscaler.internalRun(...) ~204-207
- Core relevance: bookkeeper autoscaling.
- Bug type: correctness / safety limit regression
- Proposed change: use Math.max(scaleTo, scaleUpMaxLimit) instead of Math.min.
- Trigger conditions: scale-up when cluster is near max limit.
- Expected symptom: replicas exceed configured max; resource pressure.
- Why its hard: only triggers under load; looks like misconfiguration.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with scaleUpMaxLimit and asserted clamping.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B23 - Scale-down gate uses wrong watermark
- Location: BookKeeperSetAutoscaler.checkIfCanScaleDown(...) ~229-240
- Core relevance: bookkeeper autoscaler safety checks.
- Bug type: reliability / threshold misapplication
- Proposed change: use diskUsageHwm instead of diskUsageLwm.
- Trigger conditions: disk usage normal but below HWM.
- Expected symptom: scale-down never happens; persistent overprovisioning.
- Why its hard: appears as a conservative autoscaler; not a failure.
- Static-analysis discoverability: Low.
- Suggested detection: unit test with HWM/LWM crossing scenarios.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 3/5

### B24 - Decommission picks wrong bookies
- Location: BookieDecommissionUtil.decommissionBookies(...) ~28-35
- Core relevance: bookkeeper scale-down and data movement.
- Bug type: data integrity / selection logic
- Proposed change: pick first N bookies instead of last N (or remove sorting).
- Trigger conditions: scale-down operations.
- Expected symptom: removal of newer/healthier bookies; data redistribution anomalies.
- Why its hard: only during scale-down; outcome looks like normal recovery load.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test with deterministic bookie ordering.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B25 - Decommission wait time unit mistake
- Location: BookieDecommissionUtil.decommissionBookies(...) ~53-58
- Core relevance: bookkeeper scale-down workflow.
- Bug type: time/unit mismatch
- Proposed change: Thread.sleep(3000) becomes Thread.sleep(3000 * 1000) or Thread.sleep(3).
- Trigger conditions: scale-down operations.
- Expected symptom: either long blocking (autoscaler stalls) or too-short wait (bookies not read-only yet).
- Why its hard: timing-related and intermittent.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test verifying decommission sequence timing.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B26 - Bookie admin URL ignores custom port
- Location: PodExecBookieAdminClient.computeBookieUrl(...) ~294-307
- Core relevance: bookkeeper autoscaler and decommission operations.
- Bug type: API contract drift
- Proposed change: read config using "httpServerPort" instead of "PULSAR_PREFIX_httpServerPort".
- Trigger conditions: custom bookie admin port configured.
- Expected symptom: curl to wrong port; autoscaler cannot read stats.
- Why its hard: only for customized configs.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with non-default httpServerPort.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B27 - Disk usage computed from free bytes
- Location: PodExecBookieAdminClient.parseAndFillDiskUsage(...) ~162-174
- Core relevance: bookkeeper autoscaler decisions.
- Bug type: correctness / numeric inversion
- Proposed change: set usedBytes=freeSpace or swap subtraction order.
- Trigger conditions: autoscaler uses disk usage thresholds.
- Expected symptom: scale-up/down decisions inverted.
- Why its hard: numbers still plausible; only visible under load.
- Static-analysis discoverability: Low.
- Suggested detection: unit test for disk usage parsing with known totals.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B28 - Under-replicated ledger check too strict
- Location: PodExecBookieAdminClient.doesNotHaveUnderReplicatedLedgers(...) ~276-291
- Core relevance: bookkeeper scale-down safety.
- Bug type: edge-case input handling
- Proposed change: use equals() or case-sensitive match for the BK response string.
- Trigger conditions: BK returns different wording/whitespace.
- Expected symptom: autoscaler refuses to scale down even when safe.
- Why its hard: depends on BK version output; no crash.
- Static-analysis discoverability: Low.
- Suggested detection: integration test with mocked BK response variants.
- Rankings: Exercise value 3/5; Stealth 5/5; Scorability 3/5

### B29 - Metaformat init container fails on existing clusters
- Location: BookKeeperResourcesFactory.generateStatefulSet(...) ~224-233
- Core relevance: bookkeeper pod startup.
- Bug type: reliability / upgrade regression
- Proposed change: remove "|| true" from metaformat command.
- Trigger conditions: rolling restart/upgrade on already formatted metadata.
- Expected symptom: init container fails, bookie pods never start.
- Why its hard: only on upgrade or restart, not fresh installs.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test simulating restart with existing metadata.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B30 - Journal/ledger PVC name swap
- Location: BookKeeperResourcesFactory.getJournalPvPrefix/getLedgersPvPrefix ~388-399
- Core relevance: storage naming and cleanup for bookies.
- Bug type: data integrity / storage mapping
- Proposed change: swap journal and ledgers prefixes or drop pvcPrefix.
- Trigger conditions: custom pvcPrefix or cleanup of orphan PVCs.
- Expected symptom: orphaned PVCs, data reuse across sets, or failed mounts.
- Why its hard: issues appear over time and on resizes.
- Static-analysis discoverability: Low.
- Suggested detection: integration test that asserts PVC names with prefixes.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B31 - TLS flags set when TLS disabled
- Location: BrokerResourcesFactory.patchConfigMap(...) ~189-214
- Core relevance: broker config generation.
- Bug type: API contract drift
- Proposed change: set brokerClientTlsEnabled or tls* paths unconditionally (or behind wrong flag).
- Trigger conditions: TLS disabled cluster.
- Expected symptom: brokers attempt TLS where none is configured; connection failures.
- Why its hard: configmap looks reasonable; failure appears in runtime logs.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test with TLS disabled verifying no tls* keys in broker.conf.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B32 - Plaintext ports omitted on proxy service
- Location: ProxyResourcesFactory.patchService(...) ~146-174
- Core relevance: proxy connectivity for clients.
- Bug type: correctness / service exposure
- Proposed change: require tlsEnabledOnProxy AND enablePlainTextWithTLS to add plaintext ports.
- Trigger conditions: TLS disabled or enablePlainTextWithTLS=false.
- Expected symptom: proxy service missing http/pulsar/ws ports; clients cannot connect.
- Why its hard: service is created successfully; failures only at client connection time.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test asserting service ports when TLS disabled.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B33 - Proxy broker URLs swapped
- Location: ProxyResourcesFactory.patchConfigMap(...) ~220-227
- Core relevance: proxy connects to broker using these URLs.
- Bug type: API contract drift
- Proposed change: set brokerServiceURLTLS to plaintext URL and brokerServiceURL to TLS URL.
- Trigger conditions: TLS enabled or mixed clients.
- Expected symptom: proxy fails to talk to brokers; intermittent handshake errors.
- Why its hard: values look plausible in configmap.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test validating URL values under TLS on/off.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B34 - WebSocket config uses TLS URL unconditionally
- Location: ProxyResourcesFactory.patchConfigMapWsConfig(...) ~314-319
- Core relevance: websocket proxy path.
- Bug type: correctness / API mismatch
- Proposed change: always set serviceUrl to TLS URL even when TLS disabled.
- Trigger conditions: websocket proxy enabled with plaintext broker.
- Expected symptom: websocket clients fail; only WS path impacted.
- Why its hard: standard proxy still works; failure isolated to WS.
- Static-analysis discoverability: Low.
- Suggested detection: integration test for WS proxy with TLS disabled.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

### B35 - ZooKeeper servers list off-by-one
- Location: ZooKeeperResourcesFactory.patchStatefulSet(...) ~254-257
- Core relevance: ZK ensemble config for the cluster.
- Bug type: off-by-one correctness
- Proposed change: loop for (i = 0; i <= replicas; i++) when building server list.
- Trigger conditions: any ZK replica count > 0.
- Expected symptom: ensemble config references non-existent member; quorum formation failures.
- Why its hard: looks like transient ZK instability; no compile-time hint.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test validating ZK_SERVERS length matches replicas.
- Rankings: Exercise value 4/5; Stealth 3/5; Scorability 4/5

### B36 - Superuser tokens not generated
- Location: TokenAuthProvisioner.generateSecretsIfAbsent(...) ~101-131
- Core relevance: auth bootstrap for secured clusters.
- Bug type: security / auth regression
- Proposed change: iterate proxyRoles instead of superUserRoles (or skip Base64 encoding).
- Trigger conditions: auth enabled and expects superuser tokens.
- Expected symptom: admin access fails; token secrets missing/invalid.
- Why its hard: manifests later, looks like auth configuration issue.
- Static-analysis discoverability: Medium.
- Suggested detection: integration test that verifies secrets for superuser roles.
- Rankings: Exercise value 5/5; Stealth 3/5; Scorability 4/5

### B37 - Missing DNS SANs in certs
- Location: CertManagerCertificatesProvisioner.enumerateDnsNames(...) ~298-309
- Core relevance: TLS certs for all components.
- Bug type: security / TLS compatibility
- Proposed change: drop wildcard or namespace-qualified DNS names.
- Trigger conditions: TLS enabled; clients use omitted DNS forms.
- Expected symptom: TLS verification failures for certain intra-cluster connections.
- Why its hard: only some hostnames fail; looks like intermittent TLS errors.
- Static-analysis discoverability: Low.
- Suggested detection: integration test that verifies SANs include wildcard and namespace variants.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 3/5

### B38 - CPU usage uses integer division
- Location: LoadReportResourceUsageSource.LoadReportResourceUsage.percentUsage() ~125-130
- Core relevance: broker autoscaler resource usage source.
- Bug type: numeric precision
- Proposed change: cast usage/limit to int before division.
- Trigger conditions: usage < limit with fractional ratios.
- Expected symptom: autoscaler sees 0% until usage exceeds limit; no scale-up.
- Why its hard: values still logged but rounded; only manifests under moderate load.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for percentUsage with fractional values.
- Rankings: Exercise value 4/5; Stealth 4/5; Scorability 4/5

### B39 - Migration tool drops paginated resources
- Location: migration-tool/src/main/java/com/datastax/oss/kaap/migrationtool/SpecGenerator.java, dumpOriginalResources() ~160-179
- Core relevance: migration tool diff correctness (not core operator path).
- Bug type: pagination handling
- Proposed change: list resources with a limit (ListOptions.limit) but ignore continue token.
- Trigger conditions: namespaces with many resources (server-side pagination).
- Expected symptom: diff misses resources; false positives/negatives in reports.
- Why its hard: only large clusters; output still appears valid.
- Static-analysis discoverability: Low (looks like a performance optimization).
- Suggested detection: integration test with mocked paginated list responses.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 3/5

### B40 - Deployment readiness uses reference equality
- Location: operator/src/main/java/com/datastax/oss/kaap/controllers/BaseResourcesFactory.java, isDeploymentReady(...) ~803-808
- Core relevance: readiness gating for proxy/bastion/autorecovery deployments.
- Bug type: correctness / numeric boundary
- Proposed change: replace .equals() with == when comparing availableReplicas and replicas.
- Trigger conditions: deployments with replicas > 127 (Integer cache boundary) or boxed Integer identity mismatch.
- Expected symptom: readiness check fails despite deployment being healthy; reconcile loops continue.
- Why its hard: only manifests at higher replica counts; looks like slow or stuck rollout.
- Static-analysis discoverability: Medium.
- Suggested detection: unit test for isDeploymentReady with replica count 200; integration test with large replica count.
- Rankings: Exercise value 3/5; Stealth 4/5; Scorability 4/5

## Top 10 recommended set
- B01 - Reschedule unit mismatch (core reconcile loop, time/perf regression, scorable).
- B05 - Cleanup deletes active sets (data integrity, high impact in core controllers).
- B09 - ConfigMap checksum includes resourceVersion (caching/invalidation, subtle perf bug).
- B12 - TLS scheme/port swap in service URLs (API contract drift, TLS-only failures).
- B16 - Namespace daemon tasks leak or duplicate (concurrency, nondeterministic autoscaler issues).
- B17 - Stabilization window reversed (time logic, autoscaler correctness).
- B22 - Scale-up max limit applied incorrectly (autoscaler safety boundary, high impact).
- B29 - Metaformat init container fails on existing clusters (upgrade regression, realistic).
- B36 - Superuser tokens not generated (security, auth bootstrap).
- B35 - ZooKeeper servers list off-by-one (core correctness, quorum failures).

## Instances I01–I05 (balanced by type + ranking tier)
Each instance has 8 bugs and includes a mix of correctness, reliability, time/perf, security, and data-integrity issues.
High-ranked items (from the Top 10) are distributed evenly (2 per instance).

- I01: B01, B05, B10, B18, B23, B30, B37, B39
- I02: B09, B12, B03, B14, B21, B26, B32, B40
- I03: B16, B17, B06, B11, B24, B28, B31, B38
- I04: B22, B29, B02, B07, B19, B25, B33, B34
- I05: B35, B36, B04, B08, B13, B15, B20, B27
