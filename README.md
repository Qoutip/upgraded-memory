# upgraded-memory
name: Azure - Deploy Preview Environment

# **What it does**: Build and deploy an Azure preview environment for this PR
# **Why we have it**: It's our preview environment deploy mechanism, to docs-internal and docs public repo
# **Who does it impact**: All contributors.

# !!!
# ! This worflow has access to secrets, runs in the public repository, and clones untrusted user code.
# ! Modify with extreme caution
# !!!

on:
  # The advantage of 'pull_request' over 'pull_request_target' is that we
  # can make changes to this file and test them in a pull request, instead
  # of relying on landing it in 'main' first.
  # From a security point of view, its arguably safer this way because
  # unlike 'pull_request_target', these only have secrets if the pull
  # request creator has permission to access secrets.
  pull_request_target:
    types:
      # In case someone puts the 'preview-with-translations' label on the PR
      - opened
      - synchronize
      - reopened
      - labeled
  merge_group:
  workflow_dispatch:
    inputs:
      PR_NUMBER:
        description: 'PR Number'
        type: string
        required: true
      COMMIT_REF:
        description: 'The commit SHA to build'
        type: string
        required: true
      WITH_TRANSLATIONS:
        description: 'With translations'
        required: true
        type: boolean
permissions:
  contents: read
  deployments: write

# This allows one deploy workflow to interrupt another
concurrency:
  group: 'preview-env @ ${{ github.head_ref || github.run_id }} for ${{ github.event.number || github.event.inputs.PR_NUMBER }}'
  cancel-in-progress: true

jobs:
  build-and-deploy-azure-preview:
    name: Build and deploy Azure preview environment
    runs-on: ubuntu-latest
    # Ensure this is actually a pull request and not a merge group
    # If its a merge group, report success without doing anything
    # See https://bit.ly/3qB9nZW > If a job in a workflow is skipped due to a conditional, it will report its status as "Success".
    if: |
      (
        (github.event.pull_request.head.sha || github.event.inputs.COMMIT_REF)
         && (github.event.number || github.event.inputs.PR_NUMBER || github.run_id)
       )
       && (github.repository == 'github/docs-internal' || github.repository == 'github/docs')
       &&  github.actor != 'dependabot[bot]'
    timeout-minutes: 15
    environment:
      name: preview-env-${{ github.event.number }}
      # The environment variable is computer later in this job in
      # the "Get preview app info" step.
      # That script sets environment variables which is used by Actions
      # to link a PR to a list of environments later.
      url: ${{ env.APP_URL }}
    env:
      PR_NUMBER: ${{ github.event.number || github.event.inputs.PR_NUMBER || github.run_id }}
      COMMIT_REF: ${{ github.event.pull_request.head.sha || github.event.inputs.COMMIT_REF }}
      BRANCH_NAME: ${{ github.head_ref || github.ref_name }}
      IS_INTERNAL_BUILD: ${{ github.repository == 'github/docs-internal' }}
      # This may also run in forked repositories, not just 'github/docs'
      IS_PUBLIC_BUILD: ${{ github.repository != 'github/docs-internal' }}
      NONPROD_REGISTRY_USERNAME: ${{ fromJSON('["ghdocs", "ghdocsinternal"]')[github.repository == 'github/docs-internal'] }}

    steps:
      - name: 'Az CLI login'
        uses: azure/login@92a5484dfaf04ca78a94597f4f19fea633851fa2 # pin @v1.4.6
        with:
          creds: ${{ secrets.NONPROD_AZURE_CREDENTIALS }}

      - name: 'Docker login'
        uses: azure/docker-login@83efeb77770c98b620c73055fbb59b2847e17dc0
        with:
          login-server: ${{ secrets.NONPROD_REGISTRY_SERVER }}
          username: ${{ env.NONPROD_REGISTRY_USERNAME }}
          password: ${{ secrets.NONPROD_REGISTRY_PASSWORD }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@95cb08cb2672c73d4ffd2f422e6d11953d2a9c70

      - if: ${{ env.IS_PUBLIC_BUILD == 'true' }}
        name: Check out main branch
        uses: actions/checkout@8e5e7e5ab8b370d6c329ec480221332ada57f0ab
        with:
          ref: 'main'
          persist-credentials: 'false'

      - if: ${{ env.IS_INTERNAL_BUILD == 'true' }}
        name: Check out PR code
        uses: actions/checkout@8e5e7e5ab8b370d6c329ec480221332ada57f0ab # pin @3.5.2
        with:
          ref: ${{ env.COMMIT_REF }}
          # To prevent issues with cloning early access content later
          persist-credentials: 'false'

      - name: Get preview app info
        env:
          APP_NAME_SEED: ${{ secrets.PREVIEW_ENV_NAME_SEED }}
        run: .github/actions-scripts/get-preview-app-info.sh

      - name: 'Set env vars'
        run: |
          # Image tag is unique to each workflow run so that it always triggers a new deployment
          echo "DOCKER_IMAGE=${{ secrets.NONPROD_REGISTRY_SERVER }}/${IMAGE_REPO}:${{ env.COMMIT_REF }}-${{ github.run_number }}-${{ github.run_attempt }}" >> $GITHUB_ENV

      - if: ${{ env.IS_INTERNAL_BUILD == 'true' }}
        name: Determine which docs-early-access branch to clone
        id: 'check-early-access'
        uses: actions/github-script@98814c53be79b1d30f795b907e553d8679345975
        env:
          BRANCH_NAME: ${{ env.BRANCH_NAME }}
        with:
          github-token: ${{ secrets.DOCS_BOT_PAT_READPUBLICKEY }}
          result-encoding: string
          script: |
            const { BRANCH_NAME } = process.env

            try {
              const { status } = await github.request('GET /repos/{owner}/{repo}/branches/{branch}', {
                owner: 'github',
                repo: 'docs-early-access',
                branch: BRANCH_NAME,
              })

              if (status !== 200) {
                throw new Error('Received non-200 response from branch GET request')
              }

              console.log(`Using docs-early-access branch '${BRANCH_NAME}'`)
              return BRANCH_NAME
            } catch (e) {
              console.log(`Failed to get docs-early-access branch '${BRANCH_NAME}', 'main' will be used instead.`)
              return 'main'
            }

      - if: ${{ env.IS_INTERNAL_BUILD == 'true' }}
        name: Clone docs-early-access
        uses: actions/checkout@8e5e7e5ab8b370d6c329ec480221332ada57f0ab
        with:
          repository: github/docs-early-access
          token: ${{ secrets.DOCS_BOT_PAT_READPUBLICKEY }}
          path: docs-early-access
          ref: ${{ steps.check-early-access.outputs.result }}

      - if: ${{ env.IS_INTERNAL_BUILD == 'true' }}
        name: Merge docs-early-access repo's folders
        run: .github/actions-scripts/merge-early-access.sh

      - name: Determine if we should include translations?
        uses: actions/github-script@d7906e4ad0b1822421a7e6a35d5ca353c962f410
        id: with-translations
        with:
          script: |
            if (process.env.IS_INTERNAL_BUILD !== 'true') return false
            if (context.eventName === "workflow_dispatch") {
              return context.payload.inputs.WITH_TRANSLATIONS === 'true'
            }
            // This works for pull_request_target too
            if (context.payload.pull_request?.labels) {
              return context.payload.pull_request.labels.map(label => label.name).includes('preview-with-translations')
            }
            return false

      - if: ${{ steps.with-translations.outputs.result == 'true' }}
        uses: ./.github/actions/clone-translations
        with:
          token: ${{ secrets.DOCS_BOT_PAT_READPUBLICKEY }}

      - if: ${{ env.IS_PUBLIC_BUILD == 'true' }}
        name: Check out user code to temp directory
        uses: actions/checkout@8e5e7e5ab8b370d6c329ec480221332ada57f0ab
        with:
          path: ./user-code
          ref: ${{ env.COMMIT_REF }}

      # Move acceptable user changes into our main branch checkout
      - if: ${{ env.IS_PUBLIC_BUILD == 'true' }}
        name: Move acceptable user changes
        run: |
          # Make sure recursive path expansion is enabled
          shopt -s globstar
          rsync -rptovR ./user-code/content/./**/*.md ./content
          rsync -rptovR ./user-code/assets/./**/*.png ./assets
          rsync -rptovR ./user-code/data/./**/*.{yml,md} ./data
          rsync -rptovR ./user-code/components/./**/*.{scss,ts,tsx} ./components
          rsync -rptovR --ignore-missing-args ./user-code/lib/./**/*.{js,ts} ./lib
          rsync -rptovR --ignore-missing-args ./user-code/middleware/./**/*.{js,ts} ./middleware
          rsync -rptovR ./user-code/pages/./**/*.tsx ./pages
          rsync -rptovR ./user-code/stylesheets/./**/*.scss ./stylesheets

      - uses: ./.github/actions/warmup-remotejson-cache
        with:
          restore-only: true

      # In addition to making the final image smaller, we also save time by not sending unnecessary files to the docker build context
      - name: 'Prune for preview env'
        run: .github/actions-scripts/prune-for-preview-env.sh

      - name: 'Build and push image'
        uses: docker/build-push-action@3b5e8027fcad23fda98b2e3ac259d8d67585f671
        with:
          context: .
          push: true
          target: ${{ steps.with-translations.outputs.result == 'true'  && 'production' || 'preview' }}
          tags: ${{ env.DOCKER_IMAGE }}
          # we only pull the `main` cache image
          cache-from: type=registry,ref=${{ secrets.NONPROD_REGISTRY_SERVER }}/${{ github.repository }}:main-preview
          # `main-docker-cache.yml` handles updating the remote cache so we don't pollute it with PR specific code
          cache-to: ''
          build-args: |
            BUILD_SHA=${{ env.COMMIT_REF }}

      # Succeed despite any non-zero exit code (e.g. if there is no deployment to cancel)
      - name: 'Cancel any existing deployments for this PR'
        run: |
          az deployment group cancel --name ${{ env.DEPLOYMENT_NAME }} -g ${{ secrets.PREVIEW_ENV_RESOURCE_GROUP }} || true

      # Deploy ARM template is idempotent
      # Note: once the resources exist the image tag must change for a new deployment to occur (the image tag includes workflow run number, run attempt, as well as sha)
      - name: Run ARM deploy
        uses: azure/arm-deploy@65ae74fb7aec7c680c88ef456811f353adae4d06
        with:
          resourceGroupName: ${{ secrets.PREVIEW_ENV_RESOURCE_GROUP }}
          subscriptionId: ${{ secrets.NONPROD_SUBSCRIPTION_ID }}
          template: ./azure-preview-env-template.json
          deploymentName: ${{ env.DEPLOYMENT_NAME }}
          parameters: appName="${{ env.APP_NAME }}"
            containerImage="${{ env.DOCKER_IMAGE }}"
            dockerRegistryUrl="${{ secrets.NONPROD_REGISTRY_SERVER }}"
            dockerRegistryUsername="${{ env.NONPROD_REGISTRY_USERNAME }}"
            dockerRegistryPassword="${{ secrets.NONPROD_REGISTRY_PASSWORD }}"

      - name: Check that it can be reached
        # This introduces a necessary delay. Because the preview evironment
        # URL is announced to the pull request as soon as all the steps
        # finish, what sometimes happens is that a viewer of the PR clicks
        # that link too fast and are confronted with a broken page.
        # It's because there's a delay between the `azure/arm-deploy`
        # and when the server is actually started and can receive and
        # process requests.
        # By introducing a slight "delay" here we avoid announcing a
        # preview environment URL that isn't actually working just yet.
        # Note the use of `--fail`. It which means that if it actually
        # did connect but the error code was >=400, the command will fail.
        # The `--fail --retry N` combination means that a 4xx response
        # code will exit immediately but a 5xx will exhaust the retries.
        run: curl --fail --retry-connrefused --retry 5 -I ${{ env.APP_URL }}
