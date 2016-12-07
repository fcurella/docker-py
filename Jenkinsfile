#!groovy

def imageNameBase = "dockerbuildbot/docker-py"
def imageNamePy2 = "${imageNameBase}:py2-${gitCommit()}"
def imageNamePy3 = "${imageNameBase}:py3-${gitCommit()}"
def images = [:]
def dockerVersions = ["1.12.3", "1.13.0-rc3"]

def buildImage = { name, buildargs ->
  img = docker.image(name)
  try {
    img.pull()
  } catch (Exception exc) {
    img = docker.build(name, buildargs)
    img.push()
  }
  images[name] = img
}

def buildImages = { ->
  wrappedNode(label: "ubuntu && !zfs && amd64", cleanWorkspace: true) {
    stage("build image") {
      checkout(scm)
      buildImage(imageNamePy2, ".")
      buildImage(imageNamePy3, "-f Dockerfile-py3 .")
    }
  }
}

def runTests = { Map settings ->
  def dockerVersion = settings.get("dockerVersion", null)
  def testImage = settings.get("testImage", null)

  if (!pythonVersion) {
    throw new Exception("Need test image object, e.g.: `runTests(testImage: img)`")
  }
  if (!dockerVersion) {
    throw new Exception("Need Docker version to test, e.g.: `runTests(dockerVersion: '1.12.3')`")
  }

  { ->
    wrappedNode(label: "ubuntu && !zfs && amd64", cleanWorkspace: true) {
      stage("test image=${testImage.id} / docker=${dockerVersion}") {
        checkout(scm)
        try {
          sh """docker run -d --name dpy-dind-\$BUILD_NUMBER -v /tmp --privileged \\
            dockerswarm/dind:${dockerVersion} docker daemon -H tcp://0.0.0.0:2375
          """
          sh """docker run \\
            --name dpy-tests-\$BUILD_NUMBER --volumes-from dpy-dind-\$BUILD_NUMBER \\
            -e 'DOCKER_HOST=tcp://docker:2375' \\
            --link=dpy-dind-\$BUILD_NUMBER:docker \\
            ${testImage.id} \\
            py.test -rxs tests/integration
          """
        } finally {
          sh """
            docker stop dpy-tests-\$BUILD_NUMBER dpy-dind-\$BUILD_NUMBER
            docker rm -vf dpy-tests-\$BUILD_NUMBER dpy-dind-\$BUILD_NUMBER
          """
        }
      }
    }
  }
}


buildImages()

def testMatrix = [failFast: true]

for (img in images) {
  for (version in dockerVersions) {
    testMatrix["${img.key}_${version}"] = runTests([testImage: img.value, dockerVersion: version])
  }
}

parallel(testMatrix)
