name: Android CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    name: Build & Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Gradle
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: ${{ runner.os }}-gradle

      - name: Grant Execute Permission for Gradlew
        run: chmod +x gradlew

      - name: Run Unit Tests
        run: ./gradlew testDebugUnitTest

      - name: Run Instrumentation Tests (Firebase Test Lab optional)
        run: ./gradlew connectedDebugAndroidTest

      - name: Lint Check
        run: ./gradlew lint

      - name: Static Code Analysis (SonarQube)
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          ./gradlew sonarqube \
            -Dsonar.projectKey=careconnect \
            -Dsonar.host.url=https://sonarcloud.io \
            -Dsonar.login=$SONAR_TOKEN

      - name: Generate APK
        run: ./gradlew assembleRelease

      - name: Upload Release APK
        uses: actions/upload-artifact@v3
        with:
          name: CareConnect-Release-APK
          path: app/build/outputs/apk/release/app-release.apk

  deploy:
    name: Deploy to Firebase App Distribution (Optional)
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Download APK
        uses: actions/download-artifact@v3
        with:
          name: CareConnect-Release-APK
          path: app/build/outputs/apk/release/

      - name: Deploy APK to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_APP_ID }}
          token: ${{ secrets.FIREBASE_TOKEN }}
          groups: testers
          file: app/build/outputs/apk/release/app-release.apk
