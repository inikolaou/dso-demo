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

stage('SCA') {
  steps {
    container(name: 'maven') {
      withEnv(["NVD_API_KEY=${env.NVD_API_KEY}"]) {
        sh '''
          curl -L -o dependency-check.zip https://github.com/dependency-check/DependencyCheck/releases/download/v12.1.9/dependency-check-12.1.9-release.zip
          unzip -q dependency-check.zip
          ./dependency-check/bin/dependency-check.sh \
	    --data ./dependency-check-data \
            --project "demo" \
            --scan . \
            --format "ALL" \
            --out target/dependency-check-report \
            --nvdApiKey ${NVD_API_KEY} \
	    --nvdApiDelay 6000
        '''
      }
    }
  }
  post {
    always {
      archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report/*', fingerprint: true
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

    stage('Deploy to Dev') {
      steps {
        sh 'echo done'
      }
    }

  }
}
