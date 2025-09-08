---
description: Repository Information Overview
alwaysApply: true
---

# Text-Based Fantasy Game Information

## Summary
This is a Java-based project for a text-based fantasy game. The project is in its initial stages with a basic "Hello World" application structure.

## Structure
- **src/main/java**: Contains the main application code
- **src/test/java**: Contains test files
- **.mvn**: Maven configuration directory
- **pom.xml**: Maven project configuration file

## Language & Runtime
**Language**: Java
**Build System**: Maven
**Package Manager**: Maven
**Project Type**: JAR

## Dependencies
**Main Dependencies**:
- None specified beyond the default Maven setup

**Development Dependencies**:
- junit:junit:3.8.1 - Testing framework

## Build & Installation
```bash
# Compile the project
mvn compile

# Package the application
mvn package

# Run tests
mvn test

# Clean and install
mvn clean install
```

## Testing
**Framework**: JUnit 3.8.1
**Test Location**: src/test/java/org/example
**Naming Convention**: *Test.java (e.g., AppTest.java)
**Run Command**:
```bash
mvn test
```

## Main Application
**Entry Point**: org.example.App
**Main Method**: public static void main(String[] args)
**Current Functionality**: Simple "Hello World" console output