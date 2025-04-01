# micro-services-kubernates

### Funzione per creare un nuovo servizio Spring Boot 
    function Create-SpringBootService {
    param (
        [string]$serviceName
    )
    $uri = "https://start.spring.io/starter.zip"
    $body = @{
        dependencies = "web,data-jpa,security,mysql,actuator"
        groupId = "com.example"
        artifactId = $serviceName
        name = $serviceName
        description = "$serviceName for microservices"
        packageName = "com.example.$serviceName"
        javaVersion = "17"
        springBootVersion = "3.1.0"
        type = "maven-project"
    }

    $zipFile = "$serviceName.zip"

    Invoke-WebRequest -Uri $uri -Method Post -Body $body -OutFile $zipFile
    Expand-Archive -Path $zipFile -DestinationPath $serviceName
    Remove-Item -Path $zipFile -Force

    Write-Host "✅ Servizio $serviceName creato!"
    }

    # Creiamo due nuovi microservizi
    Create-SpringBootService -serviceName "user-service"
    Create-SpringBootService -serviceName "order-service"

  ### Deployiamo i Microservizi in Kubernetes
Ora dobbiamo scrivere i file YAML per orchestrare più microservizi.

### Esegui questo script PowerShell per creare i file YAML per Kubernetes:

      # 1️⃣ Deployment per backend-service-1
      $backendDeployment = @"
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: backend-service-1
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: backend-service-1
        template:
          metadata:
            labels:
              app: backend-service-1
          spec:
            containers:
              - name: backend-service-1
                image: backend-service-1:latest
                ports:
                  - containerPort: 8080
      "@
      $backendDeployment | Out-File -FilePath "backend-service-1-deployment.yaml" -Encoding utf8
      
      # 2️⃣ Deployment per user-service
      $userServiceDeployment = @"
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: user-service
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: user-service
        template:
          metadata:
            labels:
              app: user-service
          spec:
            containers:
              - name: user-service
                image: user-service:latest
                ports:
                  - containerPort: 8081
      "@
      $userServiceDeployment | Out-File -FilePath "user-service-deployment.yaml" -Encoding utf8
      
      # 3️⃣ Deployment per order-service
      $orderServiceDeployment = @"
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: order-service
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: order-service
        template:
          metadata:
            labels:
              app: order-service
          spec:
            containers:
              - name: order-service
                image: order-service:latest
                ports:
                  - containerPort: 8082
      "@
      $orderServiceDeployment | Out-File -FilePath "order-service-deployment.yaml" -Encoding utf8

    Write-Host "✅ File YAML per i microservizi creati!"
### Ora puoi deployare tutto con:
    kubectl apply -f backend-service-1-deployment.yaml
    kubectl apply -f user-service-deployment.yaml
    kubectl apply -f order-service-deployment.yaml
### 3️⃣ Aggiungiamo un API Gateway (Spring Cloud Gateway)
  Per far comunicare i microservizi, ci serve un API Gateway.

### Creiamo un nuovo microservizio chiamato gateway-service:
    Create-SpringBootService -serviceName "gateway-service"
### Aggiungi questa dipendenza in gateway-service/pom.xml:


    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
