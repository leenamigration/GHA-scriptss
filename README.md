name: Android Management Service CI Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env: 
  OSSPI_BID: '2277' 
  OSSPI_PRODUCT_VERSION: main 
  OSSPI_RID: '000'

jobs:
  update-build-status:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Update build status on GitHub
      run: |
        curl -v POST "${{ secrets.GITHUB_WEBHOOK_URL }}" \
        --header 'Accept: application/vnd.github+json' \
        --header 'x-github-token: ${{ secrets.GITHUB_TOKEN }}' \
        --header 'Content-Type: application/json' \
        --data '{
          "event_type": "build_status",
          "client_payload": {
            "build_result_url": "https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}",
            "context": "${{ github.workflow }}",
            "commit_id": "${{ github.sha }}",
            "build_status": "InProgress",
            "build_plan_key": "${{ github.run_id }}",
            "build_number": "${{ github.run_number }}",
            "git_url": "${{ github.repository }}"
          }
        }'

  local-build:
    runs-on: ubuntu-latest
    needs: update-build-status
    env: 
      ASPNETCORE_ENVIRONMENT: Production 
      TEST_AGAINST_REMOTE: false 
      JAVA_HOME: /usr/lib/jvm/adoptopenjdk-11-hotspot
    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Inject environment variables
      run: echo "ASPNETCORE_ENVIRONMENT=Production" >> $GITHUB_ENV

    - name: Docker Cleanup
      run: ./bamboo-specs/scripts/common/docker_clean_rm.sh

    - name: Git LFS Pull
      run: ./bamboo-specs/scripts/common/lfs_pull.sh

    - name: Build and Test
      run: |
        JAVA_HOME: ${{ runner.tool_cache }}/adoptopenjdk/11
        ./build.sh --target=Default --runSonar=true --should_publish_package=true --generateApiClient=true

    - name: Chown Build Directory
      run: ./bamboo-specs/scripts/common/chown_directories.sh

    - name: Docker Cleanup
      run: ./bamboo-specs/scripts/common/docker_clean_rm.sh
    - name: Upload Log Files
      uses: actions/upload-artifact@v2
      with:
        name: log-files
        path: '**/*.log'
    - name: Upload Build Artifacts
      uses: actions/upload-artifact@v2
      with:
        name: build-artifacts
        path: artifacts/**/*
    - name: Upload Build Artifacts PB
      uses: actions/upload-artifact@v2
      with:
        name: build-artifacts-pb
        path: build/artifacts/**/*
    - name: Upload NuGet Packages PB
      uses: actions/upload-artifact@v2
      with:
        name: nuget-packages-pb
        path: build/artifacts/output/*.nupkg

  docker-build:
    runs-on: ubuntu-latest
    needs: local-build
    env:
      ASPNETCORE_ENVIRONMENT: '${{ secrets.ASPNETCORE_ENVIRONMENT }}'
      TEST_AGAINST_REMOTE: true
      JAVA_HOME: /usr/lib/jvm/adoptopenjdk-11-hotspot
      ARTIFACTORY_USERNAME: '${{ secrets.ARTIFACTORY_USERNAME }}'
      ARTIFACTORY_PASSWORD: '${{ secrets.ARTIFACTORY_PASSWORD }}'
    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Inject environment variables
        run: >
          echo "ASPNETCORE_ENVIRONMENT=${{ secrets.ASPNETCORE_ENVIRONMENT }}" >>
          $GITHUB_ENV

          echo "TEST_AGAINST_REMOTE=true" >> $GITHUB_ENV

          echo "ARTIFACTORY_USERNAME=${{ secrets.ARTIFACTORY_USERNAME }}" >>
          $GITHUB_ENV

          echo "ARTIFACTORY_PASSWORD=${{ secrets.ARTIFACTORY_PASSWORD }}" >>
          $GITHUB_ENV

    - name: Docker Cleanup
      run: ./bamboo-specs/scripts/common/docker_clean_rm.sh

    - name: Git LFS Pull
      run: ./bamboo-specs/scripts/common/lfs_pull.sh

    - name: Build Docker and Test
      run: |
        ./build.sh --target=DockerDefault --runSonar=false --publishDockerImage=true
        ./bamboo-specs/scripts/common/create_services_artifact.sh

    - name: Chown Build Directory
      run: ./bamboo-specs/scripts/common/chown_directories.sh

    - name: Docker Cleanup
      run: ./bamboo-specs/scripts/common/docker_clean_rm.sh
    - name: Upload Log Files
        uses: actions/upload-artifact@v2
        with:
          name: log-files
          path: '**/*.log'
    - name: Upload Tag
      uses: actions/upload-artifact@v2
      with:
        name: tag
        path: tag.txt
    - name: Upload Artifacts
      uses: actions/upload-artifact@v2
      with:
        name: artifacts
        path: artifact_info*.json
    - name: Upload Services Artifact
      uses: actions/upload-artifact@v2
      with:
        name: services-artifact
        path: services-artifact.zip
    - name: Upload Benchmark Artifacts
      uses: actions/upload-artifact@v2
      with:
        name: benchmark-artifacts
        path: artifacts/benchmark/**/*

  code-provenance:
    runs-on: ubuntu-latest
    needs: docker-build
    steps:
    - name: Checkout repository
      uses: actions/checkout@v2

    - name: Checkout Provenance repository
      uses: actions/checkout@v2
      with:
        repository: your-org/SRP-Provenance-Github
        path: srp-tools

    - name: Set up environment variables
        run: >
          echo "ASPNETCORE_ENVIRONMENT=Production" >> $GITHUB_ENV
          echo "TEST_AGAINST_REMOTE=true" >> $GITHUB_ENV
          echo "SRP_CLIENT_ID=${{ secrets.SRP_CLIENT_ID }}" >> $GITHUB_ENV
          echo "SRP_CLIENT_SECRET=${{ secrets.SRP_CLIENT_SECRET }}" >>
          $GITHUB_ENV

    - name: Run Provenance
      run: |
        ./Provenance.sh ${{ github.workflow }} ${{ github.run_id }} ${{ github.run_number }} ${{ secrets.SRP_CLIENT_ID }} ${{ secrets.SRP_CLIENT_SECRET }} ${{ github.sha }} ${{ github.ref }}

notifications:
  pull_request:
    types: [completed]
  webhook:
    url: http://bbs2gh.ssdevrd.com:3000/webhook
  webhook:
    url: http://ws1-build-36-61.vmware.com:8080/api/v1/dags/post_build_dag_example/dagRuns
