# How to upgrade the Gitlab Package chart
BigBang makes modifications to the upstream helm chart. The full list of changes is at the end of  this document.
1. Read release notes from upstream [Gitlab Releases](https://about.gitlab.com/releases/categories/releases/). Be aware of changes that are included in the upgrade, you can find those by [comparing the current and new revision](https://gitlab.com/gitlab-org/charts/gitlab/-/compare?from=master&to=master). Take note of any manual upgrade steps that customers might need to perform, if any.
1. Do diff of [upstream chart](https://gitlab.com/gitlab-org/charts/gitlab) between old and new release tags to become aware of any significant chart changes. A graphical diff tool such as [Meld](https://meldmerge.org/) is useful. You can see where the current helm chart came from by inspecting `/chart/Kptfile`.
1. Create a development branch and merge request from the Gitlab issue.
1. Merge/Sync the new helm chart with the existing Gitlab package code. A graphical diff tool like [Meld](https://meldmerge.org/) is useful. Reference the "Modifications made to upstream chart" section below. Be careful not to overwrite Big Bang Package changes that need to be kept. Note that some files will have combinations of changes that you will overwrite and changes that you keep. Stay alert. The hardest file to update is the `/chart/values.yaml` because the changes are many and complicated.
1. Delete all the `/chart/charts/*.tgz` files and the `/chart/requirements.lock`. You will replace these files in a later step.
1. In `/chart/requirements.yaml` update the gluon library to the latest version.
1. Run a helm dependency command to update the `chart/charts/*.tgz` archives and create a new requirements.lock file. You will commit the tar archives along with the requirements.lock that was generated.
    ```bash
    helm dependency update ./chart
    ```
1. In `/chart/values.yaml` update all the gitlab image tags to the new version. There are about 12 of them. Renovate might have already done this for you.
1. Update `/CHANGELOG.md` with an entry for "upgrade Gitlab to app version X.X.X chart version X.X.X-bb.X". Or, whatever description is appropriate.
1. Update the `/README.md` following the [gluon library script](https://repo1.dso.mil/platform-one/big-bang/apps/library-charts/gluon/-/blob/master/docs/bb-package-readme.md).
1. Update `/chart/Chart.yaml` to the appropriate versions. The annotation version should match the `appVersion`.
    ```yaml
    version: X.X.X-bb.X
    appVersion: X.X.X
    annotations:
      bigbang.dev/applicationVersions: |
        - Gitlab: X.X.X
    ```
1. Update `annotations.helm.sh/images` section in `/chart/Chart.yaml` to fix references to updated packages (if needed).
1. Use a development environment to deploy and test Gitlab. See more detailed testing instructions below. Also test with gitlab-runner to make sure it still works with the new Gitlab version. Also test an upgrade by deploying the old version first and then deploying the new version.
1. When the Package pipeline runs expect the cypress tests to fail due to UI changes. Note that most of the cypress test files are synced to the gitlab-runner Package to avoid having two different versions of the same tests. There is one place in particular that frequently fails because the button id number `button[id="__BVID__XX__BV_toggle_"]` changes in `/chart/tests/cypress/03-gitlab-login.spec.js`. It is usually necessary to run the cypress tests locally in order to troubleshoot a failing test. The following steps are about how to set up local cypress testing. There is not good documentation anywhere else so it is included here.
    1. [Install a current version of cypress](https://docs.cypress.io/guides/getting-started/installing-cypress#npm-install) on your workstation.
    1. Make a sibling directory named `cypress` next to where you have gitlab repo cloned.
        ```bash
        mkdir cypress
        ls -l
        drwxrwxr-x cypress
        drwxrwxr-x gitlab
        ```
        Inside the cypress directory create a symbolic link named `integration` that points to the cypress tests inside the gitlab repo.
        ```bash
        cd cypress
        ln -s ../gitlab/chart/tests/cypress integration
        ls -l
        lrwxrwxrwx integration -> ../gitlab/chart/tests/cypress/
        cd ..
        ```
    1. Export the environment variables that are needed by the cypress test. Reference the `bbtests:` at the end of `/chart/values.yaml`.
        ```bash
        export cypress_baseUrl=https://gitlab.bigbang.dev
        export cypress_gitlab_first_name=test
        export cypress_gitlab_last_name=user
        export cypress_gitlab_username=testuser
        export cypress_gitlab_password=12345678
        export cypress_gitlab_email=testuser@example.com
        export cypress_gitlab_project=my-awesome-project
        export cypress_adminpassword=put-the-gitlab-root-password-here
        ```
    1. Run cypress from the parent directory of the gitlab and cypress directories.
        ```bash
        cypress
        ```
    1. When Cypress launches select the same directory where you ran cypress and you should see the gitlab cypress tests listed. Run them manually, in order, one at a time.
    1. Investigate and fix errors in the cypress tests. You can run a separate browser with developer tools to find out names of elements on each page.
1. Update the `/README.md` and `/CHANGELOG.md` again if you have made any additional changes during the upgrade/testing process.

# Testing new Gitlab version
1. Create a k8s dev environment. One option is to use the Big Bang [k3d-dev.sh](https://repo1.dso.mil/platform-one/big-bang/bigbang/-/tree/master/docs/developer/scripts) with no arguments which will give you the default configuration. The following steps assume you are using the script.
1. Follow the instructions at the end of the script to connect to the k8s cluster and install flux.
1. Deploy gitlab with these dev values overrides. Core apps are disabled for quick deployment.
    ```yaml
    domain: bigbang.dev

    flux:
      interval: 1m
      rollback:
        cleanupOnFail: false

    networkPolicies:
      enabled: true

    clusterAuditor:
      enabled: false

    gatekeeper:
      enabled: false

    istioOperator:
      enabled: true

    istio:
      enabled: true

    jaeger:
      enabled: false

    kiali:
      enabled: false

    elasticsearchKibana:
      enabled: false

    eckOperator:
      enabled: false

    fluentbit:
      enabled: false

    twistlock:
      enabled: false
      values:
        console:
          persistence:
            size: 5Gi

    sso:
      oidc:
        host: login.dso.mil
        realm: baby-yoda
      client_secret: ""

    addons:

      gitlabRunner:
        enabled: true

      gitlab:
        enabled: true
        git:
          tag: null
          branch: "your-development-branch-name"

        hostnames:
          gitlab: gitlab
          registry: registry
        sso:
          enabled: true
          label: "Platform One SSO"
          client_id: "platform1_a8604cc9-f5e9-4656-802d-d05624370245_bb8-gitlab"
          client_secret: ""

        values:
          gitlab:
            webservice:
              minReplicas: 1
              maxReplicas: 1
              helmTests:
                enabled: false
            gitlab-shell:
              minReplicas: 1
              maxReplicas: 1
            sidekiq:
              minReplicas: 1
              maxReplicas: 1
          registry:
            hpa:
              minReplicas: 1
              maxReplicas: 1
          global:
            appConfig:
              object_store:
                enabled: true
              defaultCanCreateGroup: true
    ```
1. Access Gitlab UI from a browser and login with SSO.
1. Test changing your profile image.
1. In your profile create an access token with all privileges. Save the token for later use.
1. Create a group called `test`.
1. Create a project called `test1` with a README.md within the `test` group.
1. From your workstation git clone with https the test1 project.
    ```bash
    git clone https://gitlab.bigbang.dev/test/test1.git
    ```
1. Make a change to README.md and commit and push. Verify that the change shows in Gitlab UI.
1. Test pushing and pulling an image to the project container registry. Use the access token you created.
    ```bash
    docker login registry.bigbang.dev
    docker pull busybox
    docker tag busybox:latest registry.bigbang.dev/test/test1:latest
    docker push registry.bigbang.dev/test/test1:latest
    docker image rm busybox:latest
    docker image rm registry.bigbang.dev/test/test1:latest
    docker pull registry.bigbang.dev/test/test1:latest
    ```
1. Test a pipeline with gitlab-runner. Navigate to `https://gitlab.bigbang.dev/test/test1/-/settings/ci_cd` and disable the Auto DevOps. Navigate to `https://gitlab.bigbang.dev/test/test1/-/ci/editor?branch_name=main` and configure a pipeline. Verify that it completes successfully at `https://gitlab.bigbang.dev/test/test1/-/pipelines`.
   ```yaml
    stages:
      - test
    dogfood:
      stage: test
      script:
        - echo "dogfood" >> file.txt
      artifacts:
        paths:
          - file.txt
    cache:
      paths:
        - file.txt
    ```
1. Perform a manual upgrade test. First deploy the current Gitlab version. Then deploy your development branch. Verify that the upgrade is successful.
1. Retest with monitoring and logging enabled. Verify that the logging and monitoring are working.

# Modifications made to upstream chart
This is a high-level list of modifications that Big Bang has made to the upstream helm chart. You can use this as as cross-check to make sure that no modifications were lost during the upgrade process.

## chart/charts/certmanager-issuer/templates/rbac-config.yaml 
- Exposed automountServiceAccountToken for service account.
```
automountServiceAccountToken: {{ template "gitlab.serviceAccount.automountServiceAccountToken" . }}
```

## chart/charts/gitlab/charts/*/templates/serviceaccount.yaml 
- Exposed automountServiceAccountToken for service accounts in the following gitlab components: geo-logcursor, gitaly, gitlab-exporter, gitlab-pages, gitlab-shell, kas, mailroom, migrations (_serviceaccountspec.yaml), praefect, sidekiq, spamcheck, toolbox, webservice
```
automountServiceAccountToken: {{ template "gitlab.serviceAccount.automountServiceAccountToken" . }}
```

## chart/charts/gitlab/templates/_serviceAccount.tpl 
- Added template that respects the global and specific service account settings pertaining to automountServiceAccountToken
```
{{/*
Return the sub-chart serviceAccount automountServiceAccountToken setting
If that is not present it will use the global chart serviceAccount automountServiceAccountToken setting
*/}}
{{- define "gitlab.serviceAccount.automountServiceAccountToken" -}}
{{- if not (empty .Values.serviceAccount.automountServiceAccountToken) -}}
    {{ .Values.serviceAccount.automountServiceAccountToken }}
{{- else -}}
    {{ .Values.global.serviceAccount.automountServiceAccountToken }}
{{- end -}}
{{- end -}}
```

## chart/charts/nginx-ingress/values.yaml
- Added default for serviceAccount.automountServiceAccountToken in controller.admissionWebhooks to respect implicit default
```
controller:
  admissionWebhooks:
    serviceAccount:
      automountServiceAccountToken: true
```

## chart/templates/shared-secrets/job.yaml && chart/templates/shared-secrets/self-signed-cert-job.yml
- Set automountServiceAccountToken to true for shared-secrets jobs which need this token to be successful
```
automountServiceAccountToken: true
```

## chart/templates/shared-secrets/rbac-config.yaml
- Exposed automountServiceAccountToken for service account.
```
automountServiceAccountToken: {{ template "shared-secrets.automountServiceAccountToken" . }}
```

## chart/charts/registry/templates/_helpers.tpl  
- Added template that respects the global and specific service account settings pertaining to automountServiceAccountToken
```
{{/*
Return the sub-chart serviceAccount automountServiceAccountToken setting
If that is not present it will use the global chart serviceAccount automountServiceAccountToken setting
*/}}
{{- define "registry.serviceAccount.automountServiceAccountToken" -}}
{{- if not (empty .Values.serviceAccount.automountServiceAccountToken) -}}
    {{ .Values.serviceAccount.automountServiceAccountToken }}
{{- else -}}
    {{ .Values.global.serviceAccount.automountServiceAccountToken }}
{{- end -}}
{{- end -}}
```

## chart/templates/_helpers.tpl
- Added template that respects the global and specific service account settings pertaining to automountServiceAccountToken
```
{{/*
Return the sub-chart serviceAccount automountServiceAccountToken setting
If that is not present it will use the global chart serviceAccount automountServiceAccountToken setting
*/}}
{{- define "shared-secrets.automountServiceAccountToken" -}}
{{- $sharedSecretValues := index .Values "shared-secrets" -}}
{{- if not (empty $sharedSecretValues.serviceAccount.automountServiceAccountToken) -}}
    {{ $sharedSecretValues.serviceAccount.automountServiceAccountToken }}
{{- else -}}
    {{ .Values.global.serviceAccount.automountServiceAccountToken }}
{{- end -}}
{{- end -}}
```

## chart/bigbang/*
- Add DoD approved CA certificates (recursive copy directory from previous release).
- If updating new certificates from new bundle:
  - Check `Department_of_State/` certificates for spaces in name.
  - Check `DigiCert_Federal_SSP/Trust_Chain_2/` certificates for spaces in name.
  - Convert `Entrust_Federal_SSP/Trust_Chain_2/0-Entrust_Managed_Services_Root_CA_rekey3.cer` to pem format.
    ```bash
    openssl x509 -inform der -in 0-Entrust_Managed_Services_Root_CA_rekey3.cer -out 0-Entrust_Managed_Services_Root_CA_rekey3.pem
    ```
  - Remove non-certificate metadata from `Carillon_Federal_Services/Trust_Chain_1/1-Carillon_Federal_Services_PIVI_CA2.cer`.
  - Remove non-certificate metadata from `DigiCert_NFI/Trust_Chain_2/2-Senate_PIV-I_CA_G5.cer`.

## chart/templates/bigbang/*
- Add istio virtual service.
- Add networkpolicies.
- Add istio peerauthentications.
- Add Secrets for DoD certificate authorities.

## chart/templates/tests/*
- Add templates for CI helm tests.

## chart/charts/gitlab/charts/toolbox/templates/backup-job.yaml
- Added istio shutdown to command on lines 85 and 87.
  ```yaml
  {{- if and .Values.global.istio.enabled (eq .Values.global.istio.injection "enabled") }}{{ .Values.backups.cron.istioShutdown }}{{- end }}
  ```

## chart/charts/gitlab/charts/gitlab-pages/templates/service-custom-domains.yaml
- Ensure the conditional checking for empty `$externalAddresses` is removed from above the entirety of the template, and instead above the first use of it where it checks if the length of the value is `>1`. Add a closing `{{- end }}` after the existing `{{- else }}` and `{{- end }}` around the `loadBalancerIP:` & `externalIPs:` entries.
  ```yaml
  {{- if not (empty ($externalAddresses)) -}}
  {{-   if len $externalAddresses | eq 1 }}
  ...
  {{- end }}
  ```
- Remove the un-indented `{{- end }}` from the very bottom of the template (to complete the removal of the if statement being around the entire template).
- Remove the `{{- if not (empty $.Values.global.pages.externalHttp) }}` and closing `{{- end }}` from around the `80` port definition so it is always present.
- Remove the `{{- if not (empty $.Values.global.pages.externalHttps) }}` and closing `{{- end }}` from around the `443` port definition so it is always present.

## chart/charts/minio/templates/_helper_create_buckets.sh
- Hack the MinIO sub-chart to work with newer mc version in IronBank image, line 65.
  ```bash
  /usr/bin/mc policy set $POLICY myminio/$BUCKET
  ```
##  chart/charts/*.tgz
- Run `helm dependency update ./chart` and commit the downloaded archives.
- Commit the tar archives that were downloaded and requirements.lock that was generated from the helm dependency update command.

## chart/tests/*
- Add helm test scripts for CI pipeline.

## chart/templates/_certificates.tpl
- Hack to support pki certificate location within the RedHat UBI image. Is different than Debian based images. Add to definition of `gitlab.certificates.volumeMount`. The volumeMount definition is at the end of the file.
  ```yaml
  - name: etc-ssl-certs
    mountPath: /etc/pki/tls/certs/
    readOnly: true
  - name: etc-ssl-certs
    mountPath: /etc/pki/tls/cert.pem
    subPath: ca-bundle.crt
    readOnly: true
  ```

## chart/.gitignore
- Comment the `charts/*.tgz`.
- Comment the `requirements.lock`.

## chart/.helmignore
- Change `scripts/` to `/scripts/` so that the helm test scripts are not ignored.

## chart/requirements.yaml
- Add latest gluon dependency to the end of the list.
  ```yaml
  - name: gluon
    version: "x.x.x"
    repository: "oci://registry.dso.mil/platform-one/big-bang/apps/library-charts/gluon"
  ```

## chart/values.yaml
- Disable all internal services other than postgres, minio, and redis.
- Add BigBang additional values at bottom of `values.yaml`.
- Add prometheus exporter:  gitlab.gitlab-exporter.
- Add default bigbang.dev hostnames for global.hosts.
- Add IronBank hardened images.
- Add pullSecrets for each IronBank image.
- Add default bigbag.dev hostnames at global.hosts.
- Add customCAs (the cert files and secrets need to be added in the next 2 steps for this to work).
  - Run this to get a list of secrets:
  ```bash
  for i in $(helm template -s templates/bigbang/secrets/DoD_CA_certs.yaml . | grep "name:" | cut -d ":" -f 2); do echo "- secret: $i"; done
  ``````
- Add `postgresqlInitdbArgs`, `securityContext`, `postgresqlDataDir` and `persistence` to get IB image working with postgres subchart.
- Add `upgradeCheck.annotations`: sidecar.istio.io/inject: "false".
- Add `shared-secrets.annotations`: sidecar.istio.io/inject: "false".
- Add `gitlab.migrations.annotations`: sidecar.istio.io/inject: "false".
- Add `minio.jobAnnotations`: sidecar.istio.io/inject: "false".
- Add `gitlab.toolbox.annotations`: `sidecar.istio.io/proxyMemory: 512Mi` and `sidecar.istio.io/proxyMemoryLimit: 512Mi`.

# chart/Chart.yaml
- Change version key to Big Bang composite version.
- Add Big Bang `annotations.bigbang.dev/applicationVersions` and `annotations.helm.sh/images` keys to support release automation.
