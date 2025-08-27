# RAHMA
 Docker
# Build a Docker image
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- main

resources:
- repo: self

variables:
  tag: '$(Build.BuildId)'
  imageName: nodeapplication

stages:
# =========================
# Stage 1: Build & Package App
# =========================
- stage: AppBuild
  displayName: "Build and Package App"
  jobs:
  - job: Build_the_app
    pool:
      name: work
    steps:
   
    - script: |
        npm ci
      displayName: "Install dependencies"

   
    - script: |
        npm pack
        mv *.tgz $(Build.ArtifactStagingDirectory)/app.tgz
      displayName: "Packaging the app"

    
    - publish: $(Build.ArtifactStagingDirectory)/app.tgz
      artifact: app-package
      displayName: "Publish app.tgz"

# =========================
# Stage 2: Build Docker Image
# =========================
- stage: DockerImageBuild
  displayName: Build image
  dependsOn: AppBuild
  jobs:
  - job: Docker_Image
    displayName: "Build Docker Image"
    pool:
      name: work
    steps:

    - download: current
      artifact: app-package
      displayName: "Download app artifact"

  
    - task: Docker@2
      displayName: Build and Push Docker image to Docker Hub
      inputs:
        command: buildAndPush
        containerRegistry: 'bluegreen'     
        dockerfile: '$(Build.SourcesDirectory)/Dockerfile'
        buildContext: '$(Pipeline.Workspace)/app-package'
        repository: 'rahmashelbayeh/nodeapp'
        tags: $(tag)

# =========================
# Stage 3: Deploy Container
# =========================
- stage: Deploy
  displayName: "Run Container on Green VM"
  dependsOn: DockerImageBuild
  jobs:
  - job: RunContainer
    pool:
      name: blue  
    steps:
    - script: |
        echo "Checking Docker permissions..."
        docker ps || { echo 'Docker command failed. Ensure agent user is in docker group.'; exit 1; }

        echo "Stopping old container if running..."
        docker rm -f nodeapp || true

        echo "Pulling image from Docker Hub..."
        docker pull rahmashelbayeh/nodeapp:$(tag)

        echo "Starting new container..."
        docker run -d -p 80:3000 --name nodeapp rahmashelbayeh/nodeapp:$(tag) || { echo 'Container failed to start'; docker logs nodeapp; exit 1; }
      displayName: "Deploy container on Green VM"
