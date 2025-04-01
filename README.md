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

    per ogni microservizio, come user-service e order-service, dovremmo definire anche un Service in Kubernetes con un LoadBalancer se vogliamo esporli in modo appropriato e permettere loro di comunicare tra di loro e con il gateway.

### 1️⃣ Aggiungere i Service YAML per user-service e order-service
Ora, oltre ai Deployment, dobbiamo creare un file YAML per il Service di ogni microservizio, e possiamo settarlo con LoadBalancer se vogliamo esporlo all'esterno del cluster (o usare ClusterIP se i servizi devono solo comunicare tra loro internamente).

Script PowerShell per Creare i Service YAML:

        # 1️⃣ Service per user-service
        $userService = @"
        apiVersion: v1
        kind: Service
        metadata:
          name: user-service
        spec:
          selector:
            app: user-service
          ports:
            - protocol: TCP
              port: 8081
              targetPort: 8081
          type: LoadBalancer
        "@
        $userService | Out-File -FilePath "user-service-service.yaml" -Encoding utf8
        
        # 2️⃣ Service per order-service
        $orderService = @"
        apiVersion: v1
        kind: Service
        metadata:
          name: order-service
        spec:
          selector:
            app: order-service
          ports:
            - protocol: TCP
              port: 8082
              targetPort: 8082
          type: LoadBalancer
        "@
        $orderService | Out-File -FilePath "order-service-service.yaml" -Encoding utf8
        
        Write-Host "✅ File YAML per i Service di user-service e order-service creati!"
### deployare tutto con:
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
📌 1️⃣ Creiamo il file application.yml per il Gateway
Esegui questo script PowerShell per creare e configurare il file application.yml per il microservizio gateway-service:

### Crea la directory per il file di configurazione
    New-Item -ItemType Directory -Path "gateway-service/src/main/resources" -Force
    
    # Contenuto del file application.yml
    $gatewayConfig = @"
    spring:
      cloud:
        gateway:
          routes:
            - id: user-service
              uri: http://user-service:8081
              predicates:
                - Path=/users/**
            - id: order-service
              uri: http://order-service:8082
              predicates:
                - Path=/orders/**
    "@
    
    # Scrive il file application.yml
    $gatewayConfig | Out-File -FilePath "gateway-service/src/main/resources/application.yml" -Encoding utf8
    
    Write-Host "✅ File application.yml creato per il gateway!"
📌 2️⃣ Creiamo i file YAML per il Deployment e il Service del Gateway
### Ora esegui questo PowerShell per creare il Deployment di gateway-service in Kubernetes:


            # Deployment per gateway-service
            $gatewayDeployment = @"
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: gateway-service
            spec:
              replicas: 2
              selector:
                matchLabels:
                  app: gateway-service
              template:
                metadata:
                  labels:
                    app: gateway-service
                spec:
                  containers:
                    - name: gateway-service
                      image: gateway-service:latest
                      ports:
                        - containerPort: 8080
            "@
            $gatewayDeployment | Out-File -FilePath "gateway-service-deployment.yaml" -Encoding utf8

        # Service per gateway-service
        $gatewayService = @"
        apiVersion: v1
        kind: Service
        metadata:
          name: gateway-service
        spec:
          selector:
            app: gateway-service
          ports:
            - protocol: TCP
              port: 8080
              targetPort: 8080
          type: LoadBalancer
        "@
        $gatewayService | Out-File -FilePath "gateway-service-service.yaml" -Encoding utf8
        
        Write-Host "✅ File YAML per il Gateway creati!"
### Ora puoi deployare il gateway con:


        kubectl apply -f gateway-service-deployment.yaml
        kubectl apply -f gateway-service-service.yaml
📌 3️⃣ Installiamo Istio e creiamo i VirtualService
Ora abilitiamo Istio e creiamo la configurazione per orchestrare i microservizi.

Esegui questi comandi per installare Istio:

        istioctl install --set profile=demo
        kubectl label namespace default istio-injection=enabled
### Ora esegui questo script PowerShell per creare i VirtualService per ogni microservizio in Istio:

        # VirtualService per user-service
        $userVirtualService = @"
        apiVersion: networking.istio.io/v1alpha3
        kind: VirtualService
        metadata:
          name: user-service
        spec:
          hosts:
            - user-service
          http:
            - route:
                - destination:
                    host: user-service
                    port:
                      number: 8081
        "@
        $userVirtualService | Out-File -FilePath "user-service-virtualservice.yaml" -Encoding utf8

        # VirtualService per order-service
        $orderVirtualService = @"
        apiVersion: networking.istio.io/v1alpha3
        kind: VirtualService
        metadata:
          name: order-service
        spec:
          hosts:
            - order-service
          http:
            - route:
                - destination:
                    host: order-service
                    port:
                      number: 8082
        "@
        $orderVirtualService | Out-File -FilePath "order-service-virtualservice.yaml" -Encoding utf8

### VirtualService per backend-service-1
        $backendVirtualService = @"
        apiVersion: networking.istio.io/v1alpha3
        kind: VirtualService
        metadata:
          name: backend-service-1
        spec:
          hosts:
            - backend-service-1
          http:
            - route:
                - destination:
                    host: backend-service-1
                    port:
                      number: 8080
        "@
        $backendVirtualService | Out-File -FilePath "backend-service-1-virtualservice.yaml" -Encoding utf8
        
        Write-Host "✅ VirtualService YAML creati per Istio!"
### Ora puoi deployare i VirtualService con:

        kubectl apply -f user-service-virtualservice.yaml
        kubectl apply -f order-service-virtualservice.yaml
        kubectl apply -f backend-service-1-virtualservice.yaml
