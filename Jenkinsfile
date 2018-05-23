properties([
    [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', numToKeepStr: '1']],
    pipelineTriggers([
        cron('H H H * *'),
        pollSCM('H */2 * * *')
    ]),
    parameters([
        booleanParam(name: 'test', defaultValue: true, description: 'Test the image'),
        booleanParam(name: 'needAdminApproval', defaultValue: false, description: 'Wait for admin approval after testing'),
        booleanParam(name: 'release', defaultValue: true, description: 'Release the image'),
        string(name: 'logLevel', defaultValue: '3', description: 'Log level')
    ])
])

def compute_images = []
def image_version = [:]
def removable_tags = []

node {
  stage('Pull image source') {
    checkout scm
    dir('image') {
      deleteDir()
      checkout([
        $class: 'GitSCM',
        poll: false,
        branches: [[name: env.IMAGE_GIT_BRANCH ?: 'master' ]],
        userRemoteConfigs: [[url: env.IMAGE_GIT_SOURCE]]
      ])
    }
  }
  stage('Set environment') {
    manifest = readFile("${env.WORKSPACE}/image/manifest.json")
    manifest_data = new groovy.json.JsonSlurperClassic().parseText(manifest)
    image_version = manifest_data['images'][env.IMAGE_VERSION]
    env.MARKETPLACE_IMAGE_NAME = image_version['marketplace-name']
    env.IMAGE_DISK_SIZE = "50G"
    if (image_version.containsKey('options')) {
      if (image_version['options'].containsKey('disk-size')) {
        env.IMAGE_DISK_SIZE = image_version['options']['disk-size']
      }
    }
    env.BUILD_OPTS = "--pull"
    env.LOG_LEVEL = params.logLevel
  }
  stash 'image-source'
}

def image_builders = [:]
for (String arch in image_version['architectures']) {
  image_builders[arch] = stage("Create image for ${arch} on Scaleway") {
    node("${arch}&&docker") {
      unstash 'image-source'
      sh 'tree'
      withCredentials([usernamePassword(credentialsId: 'scw-test-orga-token', usernameVariable: 'SCW_ORGANIZATION', passwordVariable: 'SCW_TOKEN')]) {
        sh 'scw login -o "$SCW_ORGANIZATION" -t "$SCW_TOKEN" -s >/dev/null 2>&1'
      }
      echo "Creating image for $arch"
      withEnv(["SSH_KEY_FILE=${env.HOME}/.ssh/id_worker"]) {
        sh "make ARCH=${arch} IMAGE_DIR=${env.WORKSPACE}/image/${image_version['directory']} EXPORT_DIR=${env.WORKSPACE}/export/$arch BUILD_OPTS='${env.BUILD_OPTS}' scaleway_image"
      }
      imageId = readFile("${env.WORKSPACE}/export/${arch}/image_id").trim()
      docker_tags = readFile("${env.WORKSPACE}/export/${arch}/docker_tags").trim().split('\n')
      docker_image = docker_tags[0].split(':')[0]
      compute_images.add([
        arch: arch,
        id: imageId,
        docker_tags: docker_tags
      ])
      dir("${env.WORKSPACE}/export/${arch}") {
        sh "docker save -o docker-export-${arch}.tar ${docker_image}"
        stash "docker-export-${arch}"
        sh "docker rm ${docker_tags[-1]} && docker system prune -f"
        deleteDir()
      }
    }
  }
}
parallel image_builders

node {
  if(params.test) {
    stage('Test the images') {
      try {
        for (Map image : compute_images) {
          withEnv(["IMAGE_DIR=${env.WORKSPACE}/image/${image_version['directory']}"]) {
            sh "make tests IMAGE_DIR=${env.IMAGE_DIR} EXPORT_DIR=${env.WORKSPACE}/export/${image['arch']} ARCH=${image['arch']} IMAGE_ID=${image['id']} TESTS_DIR=${env.IMAGE_DIR}/tests NO_CLEANUP=${params.needAdminApproval}"
          }
        }
        if (env.needsAdminApproval) {
          input "Confirm that the images are stable ?"
        }
      }
      finally {
        if (env.needsAdminApproval) {
          for (Map image : compute_images) {
            sh "scripts/test_images.sh stop ${env.WORKSPACE}/export/${image['arch']}/${image['id']}.servers"
          }
        }
      }
    }
  }
}

node {
  if(params.release) {
    stage('Release the image') {
      withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASSWD')]) {
        sh 'echo -n "$DOCKERHUB_PASSWD" | docker login -u "$DOCKERHUB_USER" --password-stdin'
      }
      for (image in compute_images) {
        dir('tmp-image-extract') {
          unstash "docker-export-${image['arch']}"
          sh "docker load -i docker-export-${image['arch']}"
        }
        for (tag in image['docker_tags']) {
          sh "docker push ${tag}"
        }
        sh "docker image rm ${image['docker_tags'].join(' ')} && docker system prune -f"
        image.remove('docker_tags')
      }
      message = groovy.json.JsonOutput.toJson([
        type: "image",
        data: [
          marketplace_id: image_version['marketplace-id'],
          versions: compute_images
        ]
      ])
      versionId = input(
        message: "${message}",
        parameters: [string(name: 'image_id', description: 'ID of the new image version')]
      )
      echo "Created new marketplace version of image ${image_version['marketplace-id']}: ${versionId}"
    }
  }
}


