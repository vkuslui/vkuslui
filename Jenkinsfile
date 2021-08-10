pipeline {
    agent {
        label "master"
    }

    tools {
        maven 'maven'
        jdk 'java 11'
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "34.89.244.101:8081"
        NEXUS_REPOSITORY = "jenkins"
        NEXUS_CREDENTIAL_ID = "nexus_pass"

        postgres_ip = "13.53.122.155"
        grafana_ip = "13.51.193.67"
        tomcat_ip = "13.51.241.25"

        awx_ip = "35.242.240.88"
        jenkins_ip = "34.89.244.101"

        postgres_host = "postgres"
        tomcat_host = "tomcat"
        jenkins_host = "jenkins"
        awx_host = "awx"

    }

    stages {
        stage ('Initialize') {
            steps {
                sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
                '''
            }
        }

          stage ('Install Ansible') {
                steps {
                sh "chmod 400 aws.pem"
                sh 'yum install epel-release -y'
                sh 'yum install ansible -y'
                sh 'yum install nano -y'
                // sh 'pip install typing'
                // sh 'pip install selectors'
                // sh 'ansible-galaxy collection install community.docker'
                // sh 'ansible-galaxy collection install community.general'

                sh 'echo "[postgres]" >> /etc/ansible/hosts'
                sh "echo '${postgres_ip} ansible_user=centos ansible_ssh_private_key_file=/var/lib/jenkins/workspace/${JOB_NAME}/aws.pem' >> /etc/ansible/hosts"
                sh 'echo "[grafana]" >> /etc/ansible/hosts'
                sh "echo '${grafana_ip} ansible_user=centos ansible_ssh_private_key_file=/var/lib/jenkins/workspace/${JOB_NAME}/aws.pem' >> /etc/ansible/hosts"
                sh 'echo "[tomcat]" >> /etc/ansible/hosts'
                sh "echo '${tomcat_ip} ansible_user=centos ansible_ssh_private_key_file=/var/lib/jenkins/workspace/${JOB_NAME}/aws.pem' >> /etc/ansible/hosts"
                sh 'echo "[awx]" >> /etc/ansible/hosts'
                sh "echo '${awx_ip}' >> /etc/ansible/hosts"
                sh 'echo "[jenkins]" >> /etc/ansible/hosts'
                sh "echo '${jenkins_ip}' >> /etc/ansible/hosts"
                echo "test"
                }
          }

        stage('SSH ADD') {
           steps {
                script {
                  try {
                     sh 'ssh-keygen -b 2048 -t rsa -f ~/.ssh/id_rsa.pub -q'
                  } catch (err) {
                   echo err.getMessage()
                  }
                }
                echo currentBuild.result
                echo "test"
           }
        }

        stage('Project') {

            parallel {
                stage('Grafana') {
                    stages{
                        stage('SSH Grafana') {
                            steps {
                                 script {
                                 try {
                                    sh 'ssh -o StrictHostKeyChecking=no centos@${grafana_ip}'
                                 } catch (err) {
                             echo err.getMessage()
                                 }
                                 }
                               echo currentBuild.result
                               echo "test"
                            }
                        }

                        stage('Install Grafana') {
                            steps {
                                script {
                                     ansiblePlaybook credentialsId: 'aws_pem', playbook: 'install_grafana.yml', extras: '-e ip=http://${grafana_ip}'
                                }
                                echo "test"
                            }
                        }
                    }
                }

                stage('Postgres') {
                    stages {
                        stage('SSH Postgres') {
                            steps {
                                script {
                                     try {
                                       sh 'ssh -o StrictHostKeyChecking=no centos@${postgres_ip}'
                                   } catch (err) {
                                     echo err.getMessage()
                                     }
                                }
                                echo currentBuild.result
                                echo "test"
                            }
                        }

                        stage('Install Postgres') {
                            steps {
                                 script {
                                    ansiblePlaybook credentialsId: 'aws_pem', playbook: 'postgres_install.yml'
                                 }
                                echo "test"
                            }
                        }

                    }
                }

                stage('Build and Tomcat') {
                    stages {
                        stage ("Change localhost to Valid") {
                            steps {
                                script {
                                    sh 'sed -i "2s/localhost:8080/"$tomcat_ip":8080/"  /var/lib/jenkins/workspace/${JOB_NAME}/src/main/resources/application.properties'
                                        echo '2s string changed'
                                    sh 'sed -i "3s/localhost:8080/"$tomcat_ip":8080/" /var/lib/jenkins/workspace/${JOB_NAME}/src/main/resources/application.properties'
                                        echo '3s string changed'
                                    sh 'sed -i "26s/localhost:8080/"$tomcat_ip":8080/"  /var/lib/jenkins/workspace/${JOB_NAME}/front-end/src/main.js'
                                        echo '26s string changed'
                                    sh 'sed -i "6s/localhost:5432/"$postgres_ip":5432/" /var/lib/jenkins/workspace/${JOB_NAME}/src/main/resources/application.properties'
                                        echo '6s string changed'						
                                    sh 'sed -i "s/localhost:8080/"$tomcat_ip":8080/g"  /var/lib/jenkins/workspace/${JOB_NAME}/src/main/webapp/static/js/app.6313e3379203ca68a255.js'
                                        echo 'js changed'
                                }
                                echo "test"
                            }
                        }
                        stage('Build Project') {
                            steps {
                                script {
                                    sh 'mvn install'
                                    echo "Build SUCCESS"
                                }
                            }
                        }
                        stage("publish to nexus") {
                            steps {
                                script {
                                    pom = readMavenPom file: "pom.xml";
                                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                                    echo "target/*.${pom.packaging}";
                                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                                    artifactPath = filesByGlob[0].path;
                                    artifactExists = fileExists artifactPath;
                                    if(artifactExists) {
                                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";

                                        nexusArtifactUploader(
                                            nexusVersion: NEXUS_VERSION,
                                            protocol: NEXUS_PROTOCOL,
                                            nexusUrl: NEXUS_URL,
                                            groupId: pom.groupId,
                                            version: pom.version,
                                            repository: NEXUS_REPOSITORY,
                                            credentialsId: NEXUS_CREDENTIAL_ID,
                                            artifacts: [
                                                [artifactId: pom.artifactId,
                                                classifier: '',
                                                file: artifactPath,
                                                type: pom.packaging],

                                                [artifactId: pom.artifactId,
                                                classifier: '',
                                                file: "pom.xml",
                                                type: "pom"]
                                            ]
                                        );

                                    } else {
                                        error "*** File: ${artifactPath}, could not be found";
                                    }
                                }
                                echo "test"
                            }
                        }
                        stage('Install Tomcat') {
                            steps {
                                script {
                                    ansiblePlaybook (credentialsId: 'aws_pem', playbook: 'tomcat_install.yml', extras: '-e jenkins_ip=${jenkins_ip}')
                                    echo "test"
                                }
                            }
                        }
                    }
                }
                stage('All CollectD') {
                    stages {
                        stage('SSH Tomcat') {
                            steps {
                                script {
                                   try {
                                      sh 'ssh -o StrictHostKeyChecking=no centos@${tomcat_ip}'
                                   } catch (err) {
                                   echo err.getMessage()
                                   }
                                }
                              echo currentBuild.result
                              echo "test"
                            }
                        }
                    
                        stage('SSH AWX') {
                            steps {
                                script {
                                   try {
                                      sh 'ssh -o StrictHostKeyChecking=no centos@${awx_ip}'
                                   } catch (err) {
                                   echo err.getMessage()
                                   }
                                }
                              echo currentBuild.result
                              echo "test"
                            }
                        }

                        stage ('Install Tomcat collectd') {
                            steps {
                                sh 'sed -i "2s/.*/  hosts: ${tomcat_host}/g" /var/lib/jenkins/workspace/${JOB_NAME}/ansible_collectd/playbook.yml'
                                sh 'sed -i "16s/.*/collectd_conf_hostname: "${tomcat_host}"/g" /var/lib/jenkins/workspace/${JOB_NAME}/ansible_collectd/install-collectd/defaults/main.yml'
                                ansiblePlaybook (credentialsId: 'aws_pem', playbook: 'ansible_collectd/playbook.yml', extras: '-e grafana_ip=${grafana_ip}')
                                echo "test"
                            }
                        }

                        stage ('Install Postgres collectd') {
                            steps {
                                sh 'sed -i "2s/.*/  hosts: ${postgres_host}/g" /var/lib/jenkins/workspace/${JOB_NAME}/ansible_collectd/playbook.yml'
                                sh 'sed -i "16s/.*/collectd_conf_hostname: "${postgres_host}"/g" /var/lib/jenkins/workspace/${JOB_NAME}/ansible_collectd/install-collectd/defaults/main.yml'
                                ansiblePlaybook (credentialsId: 'aws_pem', playbook: 'ansible_collectd/playbook.yml', extras: '-e grafana_ip=${grafana_ip}')
                                echo "test"
                            }
                        }

                        stage ('Install AWX collectd') {
                            steps{
                                sh 'sed -i "2s/.*/  hosts: awx/g" /var/lib/jenkins/workspace/${JOB_NAME}/ansible_collectd/playbook.yml'
                                sh 'sed -i "16s/.*/collectd_conf_hostname: "${awx_host}"/g" /var/lib/jenkins/workspace/${JOB_NAME}/ansible_collectd/install-collectd/defaults/main.yml'
                                ansiblePlaybook (credentialsId: 'gcp_ssh', playbook: 'ansible_collectd/playbook.yml', extras: '-e grafana_ip=${grafana_ip}')
                                echo "test"
                            }
                        }

                        stage ('Install Jenkins collectd') {
                            steps{
                                sh 'sed -i "2s/.*/  hosts: 127.0.0.1\\n  connection: local/g" /var/lib/jenkins/workspace/${JOB_NAME}/ansible_collectd/playbook.yml'
                                sh 'sed -i "16s/.*/collectd_conf_hostname: "${jenkins_host}"/g" /var/lib/jenkins/workspace/${JOB_NAME}/ansible_collectd/install-collectd/defaults/main.yml'
                                ansiblePlaybook (credentialsId: 'gcp_ssh', playbook: 'ansible_collectd/playbook.yml', extras: '-e grafana_ip=${grafana_ip}')
                                echo "test"
                            }
                        }
                    }
                }
            }
        }
    }
}
