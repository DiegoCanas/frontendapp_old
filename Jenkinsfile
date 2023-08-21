pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  hostAliases:
                  - ip: "192.168.2.80"
                    hostnames:
                    - "pre.jenkins.com"
                  imagePullSecrets:
                  - name: nexus-pre
                  containers:
                  - name: jnlp
                    image: pre.docker.nexus.com/jnlp-custom:0.0.0
                  - name: alpine-core
                    image: pre.docker.nexus.com/alpine-core:0.0.0
                    command:
                    - cat
                    tty: true
            '''
        }
    }
    stages {
	stage('Ramita pa'){
		steps{
			def branch_nem = scm.branches[0].name
			if (branch_nem.contains("*/master")) {
    			branch_nem = branch_nem.split("\\*/")[1]
    			}
			else if (branch_nem.contains("*/develop"))  {
                	branch_nem = branch_nem.split("\\*/")[1]
                	}
			else  (branch_nem.contains("*/feature"))  {
                	branch_nem = branch_nem.split("\\*/")[1]
                	}
			echo branch_nem	

                }
            }
        }
    }
