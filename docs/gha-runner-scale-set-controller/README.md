# Autoscaling Runner Scale Sets mode

This new autoscaling mode brings numerous enhancements (described in the following sections) that will make your experience more reliable and secure.

## How it works

![ARC architecture diagram](arc-diagram-light.png#gh-light-mode-only)
![ARC architecture diagram](arc-diagram-dark.png#gh-dark-mode-only)

1. ARC is installed using the supplied Helm charts, and the controller manager pod is deployed in the specified namespace. A new `AutoScalingRunnerSet` resource is deployed via the supplied Helm charts or a customized manifest file. The `AutoScalingRunnerSet` controller calls GitHub's APIs to fetch the runner group ID that the runner scale set will belong to.
2. The `AutoScalingRunnerSet` controller calls the APIs one more time to either fetch or create a runner scale set in the `Actions Service` before creating the `Runner ScaleSet Listener` resource.
3. A `Runner ScaleSet Listener` pod is deployed by the `AutoScaling Listener Controller`. In this pod, the listener application connects to the `Actions Service` to authenticate and establish a long poll HTTPS connection. The listener stays idle until it receives a `Job Available` message from the `Actions Service`.
4. When a workflow run is triggered from a repository, the `Actions Service` dispatches individual job runs to the runners or runner scalesets where the `runs-on` property matches the name of the runner scaleset or labels of self-hosted runners.
5. When the `Runner ScaleSet Listener` receives the `Job Available` message, it checks whether it can scale up to the desired count. If it can, the `Runner ScaleSet Listener` acknowledges the message.
6. The `Runner ScaleSet Listener` uses a `Service Account` and a `Role` bound to that account to make an HTTPS call through the Kubernetes APIs to patch the `EphemeralRunner Set` resource with the number of desired replicas count.
7. The `EphemeralRunner Set` attempts to create new runners and the `EphemeralRunner Controller` requests a JIT configuration token to register these runners. The controller attempts to create runner pods. If the pod's status is `failed`, the controller retries up to 5 times. After 24 hours the `Actions Service` unassigns the job if no runner accepts it.
8. Once the runner pod is created, the runner application in the pod uses the JIT configuration token to register itself with the `Actions Service`. It then establishes another HTTPS long poll connection to receive the job details it needs to execute.
9. The `Actions Service` acknowledges the runner registration and dispatches the job run details.
10. Throughout the job run execution, the runner continuously communicates the logs and job run status back to the `Actions Service`.
11. When the runner completes its job successfully, the `EphemeralRunner Controller` checks with the `Actions Service` to see if runner can be deleted. If it can, the `Ephemeral RunnerSet` deletes the runner.

In addition to the increased reliability of the automatic scaling, we have worked on these improvements:

- No longer require cert-manager as a prerequisite for installing actions-runner-controller
- Reliable scale-up based on job demands and scale-down to zero runner pods
- Reduce API requests to `api.github.com`, no more API rate-limiting problems
- The GitHub Personal Access Token (PAT) or the GitHub App installation token is no longer passed to the runner pod for runner registration
- Maximum flexibility for customizing your runner pod template

### Demo

[![Watch the walkthrough](thumbnail.png)](https://youtu.be/wQ0k5k6KW5Y)

> Will take you to YouTube for a short walkthrough of the Autoscaling Runner Scale Sets mode.

## Setup

You can follow [this quickstart guide](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/quickstart-for-actions-runner-controller) for installation steps.

## Troubleshooting

You can follow [this troubleshooting guide](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/troubleshooting-actions-runner-controller-errors) for troubleshooting steps.

## Changelog

### 0.12.1

1. Fix indentation of startupProbe attributes in dind sidecar [#4126](https://github.com/actions/actions-runner-controller/pull/4126)
1. Remove duplicate float64 call [#4139](https://github.com/actions/actions-runner-controller/pull/4139)
1. Fix dind sidecar template [#4128](https://github.com/actions/actions-runner-controller/pull/4128)
1. Remove check if runner exists after exit code 0 [#4142](https://github.com/actions/actions-runner-controller/pull/4142)
1. Explicitly requeue during backoff ephemeral runner [#4152](https://github.com/actions/actions-runner-controller/pull/4152)

### 0.12.0

1. Allow use of client id as an app id [#4057](https://github.com/actions/actions-runner-controller/pull/4057)
1. Relax version requirements to allow patch version mismatch [#4080](https://github.com/actions/actions-runner-controller/pull/4080)
1. Refactor resource naming removing unnecessary calculations [#4076](https://github.com/actions/actions-runner-controller/pull/4076)
1. Fix busy runners metric [#4016](https://github.com/actions/actions-runner-controller/pull/4016)
1. Include more context to errors raised by github/actions client [#4032](https://github.com/actions/actions-runner-controller/pull/4032)
1. Revised dashboard [#4022](https://github.com/actions/actions-runner-controller/pull/4022)
1. feat(helm): move dind to sidecar [#3842](https://github.com/actions/actions-runner-controller/pull/3842)
1. Pin third party actions [#3981](https://github.com/actions/actions-runner-controller/pull/3981)
1. Fix docker lint warnings [#4074](https://github.com/actions/actions-runner-controller/pull/4074)
1. Bump the gomod group across 1 directory with 7 updates [#4008](https://github.com/actions/actions-runner-controller/pull/4008)
1. Bump go version [#4075](https://github.com/actions/actions-runner-controller/pull/4075)
1. Add job_workflow_ref label to listener metrics [#4054](https://github.com/actions/actions-runner-controller/pull/4054)
1. Bump github.com/cloudflare/circl from 1.6.0 to 1.6.1 [#4118](https://github.com/actions/actions-runner-controller/pull/4118)
1. Avoid nil point when config.Metrics is nil and expose all metrics if none are configured [#4101](https://github.com/actions/actions-runner-controller/pull/4101)
1. Bump github.com/golang-jwt/jwt/v5 from 5.2.1 to 5.2.2 [#4120](https://github.com/actions/actions-runner-controller/pull/4120)
1. Add startup probe to dind side-car [#4117](https://github.com/actions/actions-runner-controller/pull/4117)
1. Delete config secret when listener pod gets deleted [#4033](https://github.com/actions/actions-runner-controller/pull/4033)
1. Add response body to error when fetching access token [#4005](https://github.com/actions/actions-runner-controller/pull/4005)
1. Azure Key Vault integration to resolve secrets [#4090](https://github.com/actions/actions-runner-controller/pull/4090)
1. Create backoff mechanism for failed runners and allow re-creation of failed ephemeral runners [#4059](https://github.com/actions/actions-runner-controller/pull/4059)

### 0.11.0

1. Add events role permission to leader_election_role [#3988](https://github.com/actions/actions-runner-controller/pull/3988)
1. Bump github.com/golang-jwt/jwt/v4 from 4.5.1 to 4.5.2 [#3984](https://github.com/actions/actions-runner-controller/pull/3984)
1. Create configurable metrics [#3975](https://github.com/actions/actions-runner-controller/pull/3975)
1. Wrap errors in controller helper methods and swap logic in cleanups [#3960](https://github.com/actions/actions-runner-controller/pull/3960)
1. Rename log from target/actual to build/autoscalingRunnerSet version [#3957](https://github.com/actions/actions-runner-controller/pull/3957)
1. Update all dependencies, conforming to the new controller-runtime API [#3949](https://github.com/actions/actions-runner-controller/pull/3949)
1. Clean up as much as possible in a single pass for the EphemeralRunner reconciler [#3941](https://github.com/actions/actions-runner-controller/pull/3941)
1. Remove old githubrunnerscalesetlistener, remove warning and fix config bug [#3937](https://github.com/actions/actions-runner-controller/pull/3937)
1. Include custom annotations and labels to all resources created by gha-runner-scale-set chart [#3934](https://github.com/actions/actions-runner-controller/pull/3934)
1. Use Ready from the pod conditions when setting it to the EphemeralRunner [#3891](https://github.com/actions/actions-runner-controller/pull/3891)
1. Fix template tests and add go test on gha-validate-chart [#3886](https://github.com/actions/actions-runner-controller/pull/3886)
1. Update dependabot config to group packages (& include actions eco) [#3880](https://github.com/actions/actions-runner-controller/pull/3880)
1. cmd/ghalistener/config: export Validate [#3870](https://github.com/actions/actions-runner-controller/pull/3870)
1. AutoscalingRunnerSet env: not Rendering correctly [#3826](https://github.com/actions/actions-runner-controller/pull/3826)
1. Clarify syntax for githubConfigSecret [#3812](https://github.com/actions/actions-runner-controller/pull/3812)
1. Trim volume and container helpers in gha-runner-scale-set [#3807](https://github.com/actions/actions-runner-controller/pull/3807)
1. Drop verbose flag from runner scale set init-dind-externals copy [#3805](https://github.com/actions/actions-runner-controller/pull/3805)
1. Use gha-runner-scale-set-controller.chart instead of .Chart.Version [#3729](https://github.com/actions/actions-runner-controller/pull/3729)
1. metrics cardinality for ghalistener [#3671](https://github.com/actions/actions-runner-controller/pull/3671)
1. Sanitize labels ending in hyphen, underscore, and dot [#3664](https://github.com/actions/actions-runner-controller/pull/3664)
1. chore: Added OwnerReferences during resource creation for EphemeralRunnerSet, EphemeralRunner, and EphemeralRunnerPod [#3575](https://github.com/actions/actions-runner-controller/pull/3575)

### 0.10.1

1. Fix helm chart bug related to `runnerMaxConcurrentReconciles` [#3858](https://github.com/actions/actions-runner-controller/pull/3858)

### 0.10.0

This release includes major improvements to the runner provisioning duration. In short, you should see less latency between queueing a workflow run and having a runner available to execute the job.

Make sure to check [#3832](https://github.com/actions/actions-runner-controller/pull/3832) and [#3848](https://github.com/actions/actions-runner-controller/pull/3848) for details on how to fine-tune that behavior.

### Major changes

1. Add exponential backoff when generating runner reg tokens [#3724](https://github.com/actions/actions-runner-controller/pull/3724)
1. Make EphemeralRunnerController MaxConcurrentReconciles configurable [#3832](https://github.com/actions/actions-runner-controller/pull/3832)
1. Make EphemeralRunnerReconciler create runner pods earlier [#3831](https://github.com/actions/actions-runner-controller/pull/3831)
1. Make k8s client rate limiter parameters configurable [#3848](https://github.com/actions/actions-runner-controller/pull/3848)

### Minor changes

1. Bump github.com/bradleyfalzon/ghinstallation/v2 from `2.8.0` to `2.12.0` [#3837](https://github.com/actions/actions-runner-controller/pull/3837)
1. Bump golang.org/x/crypto from `0.22.0` to `0.31.0` [#3844](https://github.com/actions/actions-runner-controller/pull/3844)
1. Update docs with details for the dashboard visualizations [#3696](https://github.com/actions/actions-runner-controller/pull/3696)

### v0.9.3

1. AutoscalingListener controller: Inspect listener container state instead of pod phase [#3548](https://github.com/actions/actions-runner-controller/pull/3548)
1. Exclude label prefix propagation [#3607](https://github.com/actions/actions-runner-controller/pull/3607)
1. Check status code of fetch access token for github app [#3568](https://github.com/actions/actions-runner-controller/pull/3568)
1. Remove .Named() from the ephemeral runner controller [#3596](https://github.com/actions/actions-runner-controller/pull/3596)
1. Customize work directory [#3477](https://github.com/actions/actions-runner-controller/pull/3477)
1. Fix problem with ephemeralRunner Succeeded state before build executed [#3528](https://github.com/actions/actions-runner-controller/pull/3528)
1. Remove finalizers in one pass to speed up cleanups AutoscalingRunnerSet [#3536](https://github.com/actions/actions-runner-controller/pull/3536)

### v0.9.2

1. Refresh session if token expires during delete message [#3529](https://github.com/actions/actions-runner-controller/pull/3529)
1. Re-use the last desired patch on empty batch [#3453](https://github.com/actions/actions-runner-controller/pull/3453)
1. Extract single place to set up indexers [#3454](https://github.com/actions/actions-runner-controller/pull/3454)
1. Include controller version in logs [#3473](https://github.com/actions/actions-runner-controller/pull/3473)
1. Propogate arbitrary labels from runnersets to all created resources [#3157](https://github.com/actions/actions-runner-controller/pull/3157)

### v0.9.1

#### Major changes

1. Shutdown metrics server when listener exits [#3445](https://github.com/actions/actions-runner-controller/pull/3445)
1. Propagate max capacity information to the actions back-end [#3431](https://github.com/actions/actions-runner-controller/pull/3431)
1. Refactor actions client error to include request id [#3430](https://github.com/actions/actions-runner-controller/pull/3430)
1. Include self correction on empty batch and avoid removing pending runners when cluster is busy [#3426](https://github.com/actions/actions-runner-controller/pull/3426)
1. Add topologySpreadConstraint to gha-runner-scale-set-controller chart [#3405](https://github.com/actions/actions-runner-controller/pull/3405)

### v0.9.0

#### ⚠️ Warning

- This release contains CRD changes. During the upgrade, please remove the old CRDs before re-installing the new version. For more information, please read the [Upgrading ARC](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/deploying-runner-scale-sets-with-actions-runner-controller#upgrading-arc).
- This release contains changes in the [default docker socket path](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/deploying-runner-scale-sets-with-actions-runner-controller#upgrading-arc) expanded for container mode `dind`.
- Older version of the listener (`githubrunnerscalesetlistener`) is deprecated and will be removed in the future `0.10.0` release.

Please evaluate these changes carefully before upgrading.

#### Major changes

1. Change docker socket path to /var/run/docker.sock [#3337](https://github.com/actions/actions-runner-controller/pull/3337)
1. Update metrics to include repository on job-based label [#3310](https://github.com/actions/actions-runner-controller/pull/3310)
1. Bump Go version to 1.22.1 [#3290](https://github.com/actions/actions-runner-controller/pull/3290)
1. Propagate runner scale set name annotation to EphemeralRunner [#3098](https://github.com/actions/actions-runner-controller/pull/3098)
1. Add annotation with values hash to re-create listener [#3195](https://github.com/actions/actions-runner-controller/pull/3195)
1. Fix overscaling when the controller is much faster then the listener [#3371](https://github.com/actions/actions-runner-controller/pull/3371)
1. Add retry on 401 and 403 for runner-registration [#3377](https://github.com/actions/actions-runner-controller/pull/3377)

### v0.8.3

1. Expose volumeMounts and volumes in gha-runner-scale-set-controller [#3260](https://github.com/actions/actions-runner-controller/pull/3260)
1. Refer to the correct variable in discovery error message [#3296](https://github.com/actions/actions-runner-controller/pull/3296)
1. Fix acquire jobs after session refresh ghalistener [#3307](https://github.com/actions/actions-runner-controller/pull/3307)

### v0.8.2

1. Add listener graceful termination period and background context after the message is received [#3187](https://github.com/actions/actions-runner-controller/pull/3187)
1. Publish metrics in the new ghalistener [#3193](https://github.com/actions/actions-runner-controller/pull/3193)
1. Delete message session when listener.Listen returns [#3240](https://github.com/actions/actions-runner-controller/pull/3240)

### v0.8.1

1. Fix proxy issue in new listener client [#3181](https://github.com/actions/actions-runner-controller/pull/3181)

### v0.8.0

1. Change listener container name [#3167](https://github.com/actions/actions-runner-controller/pull/3167)
1. Fix empty env and volumeMounts object on default setup [#3166](https://github.com/actions/actions-runner-controller/pull/3166)
1. Fix override listener pod spec [#3161](https://github.com/actions/actions-runner-controller/pull/3161)
1. Change minRunners behavior and fix the new listener min runners [#3139](https://github.com/actions/actions-runner-controller/pull/3139)
1. Update user agent for new ghalistener [#3138](https://github.com/actions/actions-runner-controller/pull/3138)
1. Bump golang.org/x/oauth2 from 0.14.0 to 0.15.0 [#3127](https://github.com/actions/actions-runner-controller/pull/3127)
1. Bump golang.org.x.net from 0.18.0 to 0.19.0 [#3126](https://github.com/actions/actions-runner-controller/pull/3126)
1. Bump k8s.io/client-go from 0.28.3 to 0.28.4 [#3125](https://github.com/actions/actions-runner-controller/pull/3125)
1. Modify user agent format with subsystem and is proxy configured information [#3116](https://github.com/actions/actions-runner-controller/pull/3116)
1. Record the error when the creation pod fails [#3112](https://github.com/actions/actions-runner-controller/pull/3112)
1. Fix typo in helm chart comment [#3104](https://github.com/actions/actions-runner-controller/pull/3104)
1. Set actions client timeout to 5 minutes, add logging to client [#3103](https://github.com/actions/actions-runner-controller/pull/3103)
1. Refactor listener app with configurable fallback [#3096](https://github.com/actions/actions-runner-controller/pull/3096)
1. Bump github.com/onsi/gomega from 1.29.0 to 1.30.0 [#3094](https://github.com/actions/actions-runner-controller/pull/3094)
1. Bump k8s.io/api from 0.28.3 to 0.28.4 [#3093](https://github.com/actions/actions-runner-controller/pull/3093)
1. Bump k8s.io/apimachinery from 0.28.3 to 0.28.4 [#3092](https://github.com/actions/actions-runner-controller/pull/3092)
1. Bump github.com/gruntwork-io/terratest from 0.41.24 to 0.46.7 [#3091](https://github.com/actions/actions-runner-controller/pull/3091)
1. Record a reason for pod failure in EphemeralRunner [#3074](https://github.com/actions/actions-runner-controller/pull/3074)
1. ADR: Changing semantics of min runners to be min idle runners [#3040](https://github.com/actions/actions-runner-controller/pull/3040)

### v0.7.0

1. Add ResizePolicy and RestartPolicy on mergeListenerContainer [#3075](https://github.com/actions/actions-runner-controller/pull/3075)
1. feat: GHA controller Helm Chart quoted labels [#3061](https://github.com/actions/actions-runner-controller/pull/3061)
1. Update authorization for PAT to be Bearer as documented [#3039](https://github.com/actions/actions-runner-controller/pull/3039)
1. Metrics: set max and min runners during startup time [#3032](https://github.com/actions/actions-runner-controller/pull/3032)
1. Update Chart.yaml home URLs [#3013](https://github.com/actions/actions-runner-controller/pull/3013)
1. Remove inheritance of imagePullPolicy from manager to listeners [#3009](https://github.com/actions/actions-runner-controller/pull/3009)
1. Trim down metrics cardinality [#3003](https://github.com/actions/actions-runner-controller/pull/3003)
1. Fix role and rolebinding cleanup for the listener controller [#2970](https://github.com/actions/actions-runner-controller/pull/2970)
1. Configure listener pod with the secret instead of env [#2965](https://github.com/actions/actions-runner-controller/pull/2965)
1. Allow custom labels to be specified for controller pods [#2952](https://github.com/actions/actions-runner-controller/pull/2952)
1. Bump go version and all direct dependencies to newest for k8s compatibility [#2947](https://github.com/actions/actions-runner-controller/pull/2947)
1. chore: Service accounts in Kubernetes mode can now be annotated. [#2566](https://github.com/actions/actions-runner-controller/pull/2566)

### v0.6.1

1. Replace TLS dockerd connection with unix socket [#2833](https://github.com/actions/actions-runner-controller/pull/2833)
1. Fix name override labels when runnerScaleSetName value is set [#2915](https://github.com/actions/actions-runner-controller/pull/2915)
1. Fix nil map when annotations are applied [#2916](https://github.com/actions/actions-runner-controller/pull/2916)
1. Updates: container-hooks to v0.4.0 [#2928](https://github.com/actions/actions-runner-controller/pull/2928)

### v0.6.0

1. Fix parsing AcquireJob MessageQueueTokenExpiredError [#2837](https://github.com/actions/actions-runner-controller/pull/2837)
1. Set restart policy on the runner pod to Never if restartPolicy is not set in template [#2787](https://github.com/actions/actions-runner-controller/pull/2787)
1. Set the AutoscalingRunnerSet name to runnerScaleSetName [#2803](https://github.com/actions/actions-runner-controller/pull/2803)
1. Extend and generate crds allowing listener pod spec change [#2758](https://github.com/actions/actions-runner-controller/pull/2758)
1. Extend the user agent and fix the build version for the listener app [#2892](https://github.com/actions/actions-runner-controller/pull/2892)

### v0.5.0

1. Provide scale-set listener metrics [#2559](https://github.com/actions/actions-runner-controller/pull/2559)
1. Add DrainJobsMode [#2569](https://github.com/actions/actions-runner-controller/pull/2569)
1. Trim gha-runner-scale-set to gha-rs in names and remove role type suffixes [#2706](https://github.com/actions/actions-runner-controller/pull/2706)
1. Adapt role name to prevent namespace collision [#2617](https://github.com/actions/actions-runner-controller/pull/2617)
1. Add status check before deserializing runner-registration response [#2699](https://github.com/actions/actions-runner-controller/pull/2699)
1. Add configurable log format to values.yaml and propagate it to listener [#2686](https://github.com/actions/actions-runner-controller/pull/2686)
1. Extend manager roles to accept ephemeralrunnerset/finalizers [#2493](https://github.com/actions/actions-runner-controller/pull/2493)
1. Trim repo/org/enterprise to 63 characters in label values [#2657](https://github.com/actions/actions-runner-controller/pull/2657)
1. Revert back chart renaming [#2824](https://github.com/actions/actions-runner-controller/pull/2824)
1. Discard logs on helm chart tests [#2607](https://github.com/actions/actions-runner-controller/pull/2607)
1. Use build.Version to check if resource version is a mismatch [#2521](https://github.com/actions/actions-runner-controller/pull/2521)
1. Reordering methods and constants so it is easier to look it up [#2501](https://github.com/actions/actions-runner-controller/pull/2501)
1. chore: Set build version on make-runscaleset [#2713](https://github.com/actions/actions-runner-controller/pull/2713)
1. Fix scaling back to 0 after min runners were set to number > 0 [#2742](https://github.com/actions/actions-runner-controller/pull/2742)
1. Document customization for containerModes [#2777](https://github.com/actions/actions-runner-controller/pull/2777)
1. Bump github.com/cloudflare/circl from 1.1.0 to 1.3.3 [#2628](https://github.com/actions/actions-runner-controller/pull/2628)
1. chore(deps): bump github.com/stretchr/testify from 1.8.2 to 1.8.4 [#2716](https://github.com/actions/actions-runner-controller/pull/2716)
1. Move gha-\* docs out of preview [#2779](https://github.com/actions/actions-runner-controller/pull/2779)
1. Prepare 0.5.0 release [#2783](https://github.com/actions/actions-runner-controller/pull/2783)
1. Security fix [#2676](https://github.com/actions/actions-runner-controller/pull/2676)

### v0.4.0

#### ⚠️ Warning

This release contains a major change related to the way permissions are
applied to the manager ([#2276](https://github.com/actions/actions-runner-controller/pull/2276) and [#2363](https://github.com/actions/actions-runner-controller/pull/2363)).

Please evaluate these changes carefully before upgrading.

#### Major changes

1. Surface EphemeralRunnerSet stats to AutoscalingRunnerSet [#2382](https://github.com/actions/actions-runner-controller/pull/2382)
1. Improved security posture by removing list/watch secrets permission from manager cluster role
   [#2276](https://github.com/actions/actions-runner-controller/pull/2276)
1. Improved security posture by delaying role/rolebinding creation to gha-runner-scale-set during installation
   [#2363](https://github.com/actions/actions-runner-controller/pull/2363)
1. Improved security posture by supporting watching a single namespace from the controller
   [#2374](https://github.com/actions/actions-runner-controller/pull/2374)
1. Added labels to AutoscalingRunnerSet subresources to allow easier inspection [#2391](https://github.com/actions/actions-runner-controller/pull/2391)
1. Fixed bug preventing env variables from being specified
   [#2450](https://github.com/actions/actions-runner-controller/pull/2450)
1. Enhance quickstart troubleshooting guides
   [#2435](https://github.com/actions/actions-runner-controller/pull/2435)
1. Fixed ignore extra dind container when container mode type is "dind"
   [#2418](https://github.com/actions/actions-runner-controller/pull/2418)
1. Added additional cleanup finalizers [#2433](https://github.com/actions/actions-runner-controller/pull/2433)
1. gha-runner-scale-set listener pod inherits the ImagePullPolicy from the manager pod [#2477](https://github.com/actions/actions-runner-controller/pull/2477)
1. Treat `.ghe.com` domain as hosted environment [#2480](https://github.com/actions/actions-runner-controller/pull/2480)

### v0.3.0

#### Major changes

1. Runner pods are more similar to hosted runners [#2348](https://github.com/actions/actions-runner-controller/pull/2348)
1. Add support for self-signed CA certificates [#2268](https://github.com/actions/actions-runner-controller/pull/2268)
1. Fixed trailing slashes in config URLs breaking installations [#2381](https://github.com/actions/actions-runner-controller/pull/2381)
1. Fixed a bug where the listener pod would ignore proxy settings from env [#2366](https://github.com/actions/actions-runner-controller/pull/2366)
1. Added runner set name field making it optionally configurable [#2279](https://github.com/actions/actions-runner-controller/pull/2279)
1. Name and namespace labels of listener pod have been split [#2341](https://github.com/actions/actions-runner-controller/pull/2341)
1. Added chart name constraints validation on AutoscalingRunnerSet install [#2347](https://github.com/actions/actions-runner-controller/pull/2347)

### v0.2.0

#### Major changes

1. Added proxy support for the controller and the runner pods, see the new helm chart fields [#2286](https://github.com/actions/actions-runner-controller/pull/2286)
1. Added the abiilty to provide a pre-defined kubernetes secret for the autoscaling runner set helm chart [#2234](https://github.com/actions/actions-runner-controller/pull/2234)
1. Enhanced security posture by removing un-required permissions for the manager-role [#2260](https://github.com/actions/actions-runner-controller/pull/2260)
1. Enhanced our logging by returning an error when a runner group is defined in the values file but it's not created in GitHub [#2215](https://github.com/actions/actions-runner-controller/pull/2215)
1. Fixed helm charts issues that were preventing the use of DinD [#2291](https://github.com/actions/actions-runner-controller/pull/2291)
1. Fixed a bug that was preventing runner scale from being removed from the backend when they were deleted from the cluster [#2255](https://github.com/actions/actions-runner-controller/pull/2255) [#2223](https://github.com/actions/actions-runner-controller/pull/2223)
1. Fixed bugs with the helm chart definitions preventing certain values from being set [#2222](https://github.com/actions/actions-runner-controller/pull/2222)
1. Fixed a bug that prevented the configuration of a runner group for a runner scale set [#2216](https://github.com/actions/actions-runner-controller/pull/2216)
