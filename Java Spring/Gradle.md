
## Gradle (Kotlin DSL)

### What is Gradle
 
Gradle is a **build tool** — it compiles code, manages dependencies, runs tests, and packages the application into a JAR file.
 
You describe your project in `build.gradle.kts` (Kotlin DSL):
- what Java version to use
- what libraries to include
- how to build and test
When you run `./gradlew build`, Gradle downloads all declared dependencies from Maven Central, compiles the code, runs tests, and produces a JAR in `build/libs/`.
 
**Gradle vs Maven** — Maven is the older alternative (uses `pom.xml`). Gradle is faster, more flexible, and uses code instead of XML for configuration. Most modern Spring Boot projects use Gradle.
 

```kotlin
// build.gradle.kts
java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    compileOnly("org.projectlombok:lombok")          // compile time only
    annotationProcessor("org.projectlombok:lombok")  // required for Lombok to work
    runtimeOnly("org.postgresql:postgresql")          // only needed at runtime
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

**Dependency scopes:**
- `implementation` — available everywhere
- `compileOnly` — only during compilation, not in final JAR
- `runtimeOnly` — only at runtime, not during compilation
- `annotationProcessor` — runs during compilation to generate code (Lombok)
- `testImplementation` — only in tests

> Use `sourceCompatibility` / `targetCompatibility` instead of `toolchain` to avoid Gradle JDK resolution issues when using SDKMAN.

---

### Project structure (feature-based)

```
task_manager/
├── auth/
│   ├── AuthController.java
│   ├── AuthService.java
│   ├── JwtFilter.java
│   ├── JwtService.java
│   └── dto/
├── config/
│   ├── SecurityConfig.java
│   └── OpenApiConfig.java
├── task/
│   ├── Task.java
│   ├── TaskController.java
│   ├── TaskRepository.java
│   ├── TaskService.java
│   ├── TaskStatus.java
│   └── dto/
├── user/
│   └── ...
└── category/
    └── ...
```
