// Token para kubernetes

/*
@Grab(group='io.fabric8', module='kubernetes-client', version='5.0.0')
import io.fabric8.kubernetes.api.model.Namespace
import io.fabric8.kubernetes.client.*
def tokensito = sh 'kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')'

def clusterUrl = 'https://tu-cluster-k8s-url'  // Reemplaza con la URL de tu clúster Kubernetes
def token = ${tokensito}  // Reemplaza con el token de autenticación

def config = new ConfigBuilder()
    .withMasterUrl(clusterUrl)
    .withOauthToken(token)
    .withTrustCerts(true)  // Usar si el servidor Kubernetes utiliza certificados autofirmados
    .build()

def client = new DefaultKubernetesClient(config)
*/

def IMAGE_NAME = dockerfile  // Meter el nombre del repositorio // Se saca de variable de entorno
def TAG_TO_CHECK = nextTag()
def PREVIOUS_TAG = lastTag()
String NEXUS_REGISTRY_URL = 'pre.docker.nexus.com'

// Ahora puedes usar 'client' para interactuar con el clúster Kubernetes
def namespaces = client.namespaces().list()
namespaces.items.each { Namespace ns ->
    println("Namespace: ${ns.metadata.name}")
}





//Se genera el mapa con la configuración
Map config = [
    //Unos corhcetes {} significan que vas a meter un trozo de código. Como si ejecutases algo dentro del if
    branchType: {
        if (env.GIT_BRANCH_NAME == 'master') {
            return 'MASTER'
        }
        else if (env.GIT_BRANCH_NAME ~== 'feature.*') {
            return 'FEAT'
        }
        else if (env.GIT_BRANCH_NAME ~== 'break.*') {
            return 'BREAK'
        }
        else if (env.GIT_BRANCH_NAME ~== 'fix.*') {
            return 'FIX'
        }
        else {
            return 'UKNOWN'
        }
    },
        // Url del repo, https://stackoverflow.com/questions/45937337/jenkins-pipeline-get-repository-url-variable-under-pipeline-script-from-scm
        httpRepoUrl: scm.userRemoteConfigs[0].url,
        githubToken : {
        withCredentials([usernameColonPassword(credentialsId:'idSecretoJenkins', variable : 'GITHUB_TOKEN')]) {
            return $GITHUB_TOKEN


        // Kubernetes
        sh 'apk add kubectl'
        // Helm
        sh 'apk add helm'
        // Docker
        sh 'apk install docker'
        sh 'rc-update add docker boot'
        sh 'service docker start'
        // npm
        sh 'apk add nodejs npm'

        // Kubeconfig
        KUBECONFIG_CREDENTIALS = credentials('carne')
        }
    }
]

// Mirar de reemplazar con findFile
def buscarArchivo(String nombre_ms, String expresion)
{
    ls = sh(script: "ls ./${nombre_ms}", returnStdout: true).trim()
    String[] archivos =  ls.split("\\s+"); 
    String out = "";
    for(int i = 0; i < archivos.length; ++i)
    {
        if(archivos[i].contains(expresion))
        {
            out = archivos[i];
        }
    }
    return out;
}

/*
def isHelmInstalled() {
    def process = 'helm version --short'.execute()
    process.waitForOrKill(3000) // Esperar como máximo 3 segundos para la respuesta

    if (process.exitValue() == 0) {
        // La ejecución de "helm version" fue exitosa, lo que indica que Helm está instalado.
        return true
    } else {
        // La ejecución de "helm version" falló, lo que indica que Helm no está instalado.
        return false
    }
}*/

pipeline{
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
                    - "pre.gitea.com"
                  imagePullSecrets:
                  - name: nexus-pre
                  containers:
                  - name: jnlp
                    image: pre.docker.nexus.com/jnlp-custom:0.0.0
                    env:
                    - name: GIT_SSL_NO_VERIFY
                      value: true
                    - name: JSL_GIT_EMAIL
                      value: 'mmll_bot_gh@protonmail.com'
                    - name: JSL_GIT_REPO
                      value: 'https://pre.gitea.com/ValentiaSoft_DevOps/jsl-conf.git'
                    - name: JSL_GIT_BRANCH
                      value: 'feature/versionado'
                    - name: JSL_CONF_PATH
                      value: 'jsl-config.yaml'
                    - name: JSL_GITEA_CREDENTIALS_ID
                      value: 'gitea-devops-bot'
                  - name: alpine-core
                    image: pre.docker.nexus.com/alpine-core:0.0.0
                    env:
                    - name: GIT_SSL_NO_VERIFY
                      value: true
                    command:
                    - cat
                    tty: true
            '''
        }
    }
    stages {
        stage('Get next version') {
            steps {
                script {
                    if (!(isValidBranch() && isPullRequestToMaster())) {
                        error('Is not a pull request')
                        currentBuild.result = "SUCCESS"
                    }
                 }
            }
            stage('Kubeconfig') {
                //Has de contruir el proyecto de front para que el Dockerfile lo recoja
                steps {
                    script {
                        if (isValidBranch()) {
                            //Se hace desde el docker todo
                        }
                    }
                }
            }
            /* Te los dejo creados a mano
            stage('Crear Namespaces') {
                steps {
                    script {
                        def kubeconfig = '/ruta/kubeconfig'
                            // ¿Plantear simular la rama a producción como herramienta de CD con otra pipeline?
                            sh "kubectl --kubeconfig=${kubeconfig} create namespace produccion"
                            sh "kubectl --kubeconfig=${kubeconfig} create namespace preproduccion"
                        }
                    }
                }
            }
            */
            stage('Docker') {
                steps {
                    script {
                        if (isValidBranch()) {
                            String dockerfile = buscarArchivo( app, "Dockerfile")
                            if(dockerfile != "")
                            {
                                String tag = calculateNextTag()
                                createTag(tag)

                                // Push a Nexus
                                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                                    sh ("""
                                        docker image build -f ./${app}/${dockerfile} -t ${app}:${tag}                                        
                                        docker login -u ${NEXUS_USERNAME} -p ${NEXUS_PASSWORD} ${NEXUS_REGISTRY_URL}
                                        docker tag ${app}:${tag} ${NEXUS_REGISTRY_URL}/${app}:${tag}
                                        docker push ${NEXUS_REGISTRY_URL}/${dockerfile}:${tag}
                                    """)
                                }
                            }
                            else
                            {
                                currentBuild.result = "FAILURE"
                                throw new Exception("No existe ningun Dockerfile en el repositorio.") // Reemplazar por lo de abajo
                                //error('No existe ningun Dockerfile en el repositorio')
                            }
                        }
                        else{
                            currentBuild.result = "FAILURE"
                            throw new Exception("El microservicio ${NOMBRE_MS} no existe en el directorio de GitHub.")
                        }
                    }
                }
            }
            stage('Helm') {
                //Implementar quí despliegue de Helm
                steps {
                    script {
                        if (isValidBranch()) {
                            if (isHelmInstalled()) {
                                def helmInstalled = (script: 'helm list -a', returnStatus: true, returnStdout: true).trim()
                                if branchType == 'Master'{
                                    if (helmInstalled.contains('Variable de entorno que ha chupado el hhtps')){
                                        echo 'instalado frontendapp'
                                    }
                                    else {
                                        sh script : "helm --kubeconfig=${kubeconfig} install mi-release mi-chart --namespace=produccion -f values-produccion.yaml"
                                    }   
                                }
                                else {
                                //sh script : "helm --kubeconfig=${kubeconfig} install mi-release mi-chart --namespace=preproduccion -f values-preproduccion.yaml"
                            }
                            //Investigar 'helm list -a'
                            //Investigar que es el 'helm upgrade'
                        }
                    }
                }
            }
        }
        post {
            always {
                // Limpieza o acciones finales que deben realizarse sin importar el resultado
            }
            success {
                echo "Pipeline completed successfully"
            }
            failure {
                echo "Pipeline failed"
                if (dockerfileContents.contains("LABEL version=\"$TAG_TO_CHECK\"")){
                // Agregar aquí acciones adicionales en caso de que el pipeline falle
                // Falta comprobacion de si se va a hacer el rollback o no
                //Cambiamos etiqueta a la anterior
                sh "docker tag $IMAGE_NAME:$TAG_TO_CHECK $IMAGE_NAME:$PREVIOUS_TAG"
                // Fuera imagen + etiqueta // Tener en cuenta que se borra la imagen de nexus, eso hace la local
                sh "docker rmi $IMAGE_NAME:$TAG_TO_CHECK"
                curl -X DELETE -u admin:admin123  "http://somedomain/nexus/content/repositories/myrepo/com/test/test-artifact/1.0.0/"
                else{
                    echo 'Etiqueta no cambiada'
                    curl -X DELETE -u admin:admin123  "http://somedomain/nexus/content/repositories/myrepo/com/test/test-artifact/1.0.0/"
                }
                }
            }
        }
    }
}

boolean isValidBranch() { // Triggers
    return config.branchType != 'UKNOWN'
}

boolean isFeature() {
    return config.branchType == 'FEAT'
}

boolean isBreak() {
    return config.branchType == 'BREAK'
}

boolean isFix() {
    return config.branchType == 'FIX'
}

boolean isMaster() {
    return config.branchType == 'MASTER'
}

boolean isPullRequestToMaster() {
    return env.CHANGE_TARGET == 'refs/heads/master'
}

String lastTag() {
    /*
        El sh ejecuta sentencias bash
        Obten los tags remotos
    */
    sh('git ls-remote --tags origin')
    
    //Ordenalos alfanuméricamente para obtener el último
    return sh(script: 'git describe --tags --abbrev=0', stdout : true)
}

String calculateNextTag() {


    // https://www.baeldung.com/groovy-convert-string-to-integer
    
    def tagParts = lastTag().tokenize('.')  // tokenize quita espacios
    def x = tagParts[0] as int
    def y = tagParts[1] as int
    def z = tagParts[2] as int
    
    //Esta parte te la dejo a ti, quiero dormir (ten en cuenta si no hay tags en el repo)
    if (isFeature()) {
        y++
    }
    else if (isBreak()) {
        x++
    }
    else if (isFix()) {
        z++
    } else if(isMaster()){
        //Decidir que hacer
    }
    
    return "${x}.${y}.${z}"
}

void createTag(String nextTag) {
    //withCredentials()...  investigar cómo logarse en git

    echo("Siguiente version calculada: ${nextTag}")
    sh("git tag ${nextTag}")
    sh("git push origin ${nextTag}")
}