pipeline {
    agent any

    stages {
        stage ('Artifactory Configuration') {
            steps {
                rtMavenResolver (
                    id: 'maven-resolver',
                    serverId: 'talyi-artifactory',
                    releaseRepo: 'libs-release',
                    snapshotRepo: 'libs-snapshot'
                )  
                 
                rtMavenDeployer (
                    id: 'maven-deployer',
                    serverId: 'talyi-artifactory',
                    releaseRepo: 'libs-release-local',
                    snapshotRepo: 'libs-snapshot-local',
                    threads: 6,
                    properties: ['BinaryPurpose=Technical-BlogPost', 'Team=DevOps-Acceleration']
                )
            }
        }
        
        stage('Build Maven Project') {
            steps {
                rtMavenRun (
                    tool: 'Maven 3.3.9',
                    pom: 'pom.xml',
                    goals: '-U clean install',
                    deployerId: "maven-deployer",
                    resolverId: "maven-resolver"
                )
            }
        }

        stage ('Publish build info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "talyi-artifactory"
                )
            }
        }

        stage ('Xray Maven Scan') {
            steps {
                xrayScan (
                    serverId: "talyi-artifactory",
                    failBuild: true
                )
            }
        } 

        stage ('Promotion') {
            steps {
                rtPromote (
                    serverId: "talyi-artifactory",
                    targetRepo: 'libs-release-local',
                    comment: 'Passed Xray QualityGate',
                    status: 'Released',
                    includeDependencies: false,
                    failFast: true,
                    copy: true
                )
            }
        }

        stage ('Build Docker Image') {
            steps {
                script {
                    docker.build("talyi-docker.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}")
                }
            }
        }

        stage ('Push Image to Artifactory') {
            steps {
                rtDockerPush(
                    serverId: "talyi-artifactory",
                    image: "talyi-docker.jfrog.io/" + "pet-clinic:1.0.${env.BUILD_NUMBER}",
                    targetRepo: 'docker',
                    properties: 'project-name=jfrog-blog-post;status=stable'
                )
            }
        }

        stage ('Publish Build Info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "talyi-artifactory"
                )
            }
        }

        stage ('Xray Docker Image Scan') {
            steps {
                xrayScan (
                    serverId: "talyi-artifactory",
                    failBuild: true
                )
            }
        }         
        
        stage ('Install JFrog CLI') {
            steps {
                 sh '''
                    curl -fL https://getcli.jfrog.io | sh
                 '''
            }
        }         

        stage ('Create & Sign Release Bundle') {
            steps {
                 sh '''
                    jfrog rt rbc --spec=RB-spec.json --sign EU-LISA-RB 1.0.${env.BUILD_NUMBER}
                 '''
            }
        }

        stage ('Export Release Bundle') {
            steps {
                 sh '''
                    curl -XPOST 'https://talyi.jfrog.io/distribution/api/v1/export/release_bundle/EU-LISA-RB/1.0.${env.BUILD_NUMBER}' -uadmin:password
                 '''
            }
        }
    }
}
