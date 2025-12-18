pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }

  environment {
      NVD_API_KEY       = credentials('jenkins-nvd-api-key')
      OSSINDEX_USERNAME = credentials('jenkins-oss-username')
      OSSINDEX_PASSWORD = credentials('jenkins-oss-token')
  }

  stages {
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container(name: 'maven') {
              sh 'mvn compile'
            }

          }
        }

      }
    }

    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container(name: 'maven') {
              sh 'mvn test'
            }
          }
        }

	stage('Generate SBOM') {
	  steps {
	    container(name: 'maven') {
	      sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
	    }
	  }
	  post {
	    success {
	      // dependencyTrackPublisher projectName: 'sample-spring-app', projectVersion: '0.0.1', artifact: 'target/bom.xml', autoCreateProjects: true, synchronous: true
	      archiveArtifacts allowEmptyArchive: true, artifacts: 'target/bom.xml', fingerprint: true, onlyIfSuccessful: true
	    }
	  }
	}

	stage('OSS License Checker') {
          steps {
            container(name: 'licensefinder') {
                sh 'ls -al'
                sh '''#!/bin/bash --login
                        /bin/bash --login
                        rvm use default
                        gem install license_finder
                        license_finder
                    '''
            }
          }
        }
      }
    }

    stage('SAST') {
	steps {
	  container(name: 'slscan') {
	    sh 'scan --type java,depscan --build'
	  }
	}
	post {
	  success {
	    archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/*', fingerprint: true, onlyIfSuccessful: true
	  }
	}
    }

    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container(name: 'maven') {
              sh 'mvn package -DskipTests'
            }

          }
        }

	stage('OCI Image BnP') {
	  steps {
	    container(name: 'kaniko') {
	      sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --destination=docker.io/inikolaou/dso-demo'
	    }
	  }
	}
      }
    }

    stage('Image Analysis') {
      parallel {
	stage('Image Linting') {
	  steps {
	    container(name: 'docker-tools') {
	      sh 'dockle docker.io/inikolaou/dso-demo'
	    }
	  }
	}
	stage('Image Scan') {
	  steps {
	    container(name: 'docker-tools') {
	      sh 'trivy image --timeout 10m --exit-code 1 inikolaou/dso-demo'
	    }
	  }
	}
      }
    }

    stage('Deploy to Dev') {
      steps {
        sh 'echo done'
      }
    }

  }
}
