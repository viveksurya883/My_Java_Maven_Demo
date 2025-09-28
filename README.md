CI/CD Pipeline Demo — Java Maven with AKS Deployment

This project demonstrates a complete CI/CD workflow for a Java Maven application, showcasing industry-standard tools and best practices. The pipeline takes code from source to production automatically, ensuring quality, traceability, and scalability.
📌 Workflow Overview
1. Code Commit → GitHub Actions
2. Maven Build
3. SonarQube Quality Check
4. Artifact Management (JFrog Artifactory)
5. Dockerization
6. Deployment on Azure Kubernetes Service (AKS)


⚙️ Tools & Technologies

- CI/CD → GitHub Actions
- Build → Maven
- Code Quality → SonarQube
- Artifact Storage → JFrog Artifactory
- Containerization → Docker
- Deployment → Azure Kubernetes Service (AKS)


🏗️ Pipeline Implementation

1️⃣ Initialization
mvn archetype:generate
2️⃣ Build & Test
mvn clean install
Output → target/my_java_maven_demo-0.0.1-SNAPSHOT.jar
3️⃣ Code Quality with SonarQube
mvn sonar:sonar \
  -Dsonar.projectKey=my-java-maven-demo \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.login=$SONAR_TOKEN
4️⃣ Artifact Upload to JFrog
jf rt upload target/*.jar maven-local/com/example/my_java_maven_demo/0.0.1-SNAPSHOT/
5️⃣ Docker Build & Push
docker build -t docker.io/<username>/my_java_maven_demo:<commit_sha> .
docker push docker.io/<username>/my_java_maven_demo:<commit_sha>
6️⃣ Kubernetes Deployment on AKS
deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-java-maven-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-maven-demo
  template:
    metadata:
      labels:
        app: java-maven-demo
    spec:
      containers:
      - name: java-maven-demo
        image: docker.io/<username>/my_java_maven_demo:<commit_sha>
        ports:
        - containerPort: 8080
service.yaml
apiVersion: v1
kind: Service
metadata:
  name: java-maven-service
spec:
  type: LoadBalancer
  selector:
    app: java-maven-demo
  ports:
    - port: 80
      targetPort: 8080
      
Deploy to AKS:
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml


🔎 Verification

Check running pods:
 kubectl get pods
Check LoadBalancer service:
 kubectl get svc java-maven-service
Example output:
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
java-maven-service   LoadBalancer   10.0.123.45    52.176.xxx.xxx   80:32000/TCP   2m
Visit → http://4.187.180.12/


📊 CI/CD Workflow Diagram

```mermaid
flowchart LR
    A[Developer Commit] --> B[GitHub Actions]
    B --> C[Maven Build & Test]
    C --> D[SonarQube Analysis]
    D --> E[JFrog Artifactory (Store JAR)]
    E --> F[Docker Build & Push to Docker Hub]
    F --> G[Azure Kubernetes Service]
    G --> H[Pods Running in AKS]
    H --> I[LoadBalancer Service]
    I --> J[User Access via Browser]
```




<!-- last successful commit ci.yml: this uses github actions to upload the artifact and then downloads from the docker for image creation.

name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  IMAGE_NAME: My_Java_Maven_Demo

jobs:
  # -------------------- INIT --------------------
  init:
    runs-on: ubuntu-latest
    outputs:
      m2-cache-key: ${{ steps.cache-m2.outputs.cache-hit }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up JDK 17 and cache Maven
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
          cache: maven

      - name: Cache ~/.m2/repository
        id: cache-m2
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-m2-

      - name: Create Maven settings.xml (for Artifactory)
        run: |
          mkdir -p ~/.m2
          cat > ~/.m2/settings.xml <<EOF
          <settings>
            <servers>
              <server>
                <id>artifactory-repo</id>
                <username>${{ secrets.ARTIFACTORY_USER }}</username>
                <password>${{ secrets.ARTIFACTORY_PASSWORD }}</password>
              </server>
            </servers>
          </settings>
          EOF

  # -------------------- BUILD, TEST & PACKAGE --------------------
  build-package:
    runs-on: ubuntu-latest
    needs: init
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17 and cache Maven
        uses: actions/setup-java@v4
        with:
            distribution: temurin
            java-version: 17
            cache: maven

      - name: Build & Test & Package (Maven)
        run: mvn -B clean verify package -DskipTests=false

      - name: Upload JAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

        # -------------------- SONARQUBE --------------------
  sonarqube:
    runs-on: ubuntu-latest
    needs: build-package
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17 and cache Maven
        uses: actions/setup-java@v4
        with:
            distribution: temurin
            java-version: 17
            cache: maven

      - name: Run SonarQube Analysis
        run: |
         mvn -B verify sonar:sonar \
         -Dsonar.projectKey=viveksurya883_My_Java_Maven_Demo \
         -Dsonar.organization=viveksurya883 \
         -Dsonar.host.url=https://sonarcloud.io \
         -Dsonar.login=${{ secrets.SONAR_TOKEN }}


  # -------------------- DOCKER --------------------
  docker:
    runs-on: ubuntu-latest
    needs: sonarqube
    steps:
      - uses: actions/checkout@v4

      - name: Download JAR artifact
        uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: target

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Docker image
        run: docker build -t docker.io/${{ secrets.DOCKER_USERNAME }}/my_java_maven_demo:latest .

      - name: Push Docker image
        run: docker push docker.io/${{ secrets.DOCKER_USERNAME }}/my_java_maven_demo:latest


  # -------------------- ARTIFACTORY --------------------
  publish-artifact:
    runs-on: ubuntu-latest
    needs: sonarqube
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17 and cache Maven
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
          cache: maven

      - name: Download JAR artifact
        uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: target

      - name: Install JFrog CLI
        uses: jfrog/setup-jfrog-cli@v2
        with:
         version: latest

      - name: Configure JFrog CLI
        run: |
         jf c add my-artifactory \
         --url=https://trialz1c1tf.jfrog.io \
         --user="${{ secrets.ARTIFACTORY_USER }}" \
         --password="${{ secrets.ARTIFACTORY_PASSWORD }}" \
         --interactive=false

      - name: Upload artifact (JAR)
        run: jf rt upload "target/*.jar" "maven-local/my_java_maven_demo/${{ github.run_number }}-${{ github.sha }}/" --flat=true --server-id=my-artifactory -->



